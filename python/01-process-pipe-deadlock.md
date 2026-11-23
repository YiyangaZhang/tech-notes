# 01 Python 多进程：从死锁到可靠结果传输

> 原稿最早可见 Git 记录：2026-10-14；原稿最近 Git 修改：2026-10-15。公开版整理：2026-11-17。以上 Git 日期用于追溯，不等同于实际首次写作日期。
> 本文为通用知识整理，历史产品能力及版本细节需结合当前文档复核。

章节编号沿用原稿，便于在系列中回溯；长代码与推导可折叠阅读。

## 从死锁到可靠实现

下面先说明执行顺序为什么会导致循环等待。已经熟悉 `start()`、`join()` 和 `recv()` 的读者可以直接跳到“推荐实现”。

<details>
<summary>原理推导：进程调度、消息传输与死锁时序（第 3–8 节）</summary>

### 3. 首先理解：`start()` 不是普通函数调用

普通函数调用：

```python
html = child_function()
print(html)
```

执行关系是单一控制流：

```text
父进程进入 child_function()
        ↓
函数执行完成并返回
        ↓
父进程继续执行 print(html)
```

而多进程代码：

```python
process.start()
```

会让操作系统启动另一个进程。`start()` 返回以后，父进程和子进程成为两个可以并发推进的执行流：

```text
时间 ───────────────────────────────────────────────>

父进程：process.start() ── 下一条语句 ── 下一条语句
                    │
                    └──── 创建或启动子进程

子进程：                 target() ── generate_html() ── send()
```

父进程启动子进程，不代表父进程必须等子进程完成后才能继续。父进程下一步是否等待，以及等待什么，完全取决于 `start()` 后面的代码。Python 官方文档说明，`Process.start()` 启动进程，而 `Process.join()` 用于等待进程终止。[Python `multiprocessing` 文档](https://docs.python.org/3/library/multiprocessing.html)

操作系统调度器可能交替运行两者，也可能让其中一个连续运行一段时间。正确性不能依赖“我猜父进程会先运行”或“我猜子进程会先完成”，而必须依赖明确的同步和通信设计。


### 4. `join()` 和 `recv()` 都会等待，但等待的不是同一件事

#### 4.1 `join()` 等待子进程终止

```python
process.join()
```

其语义可以概括为：

```text
暂停当前父进程，直到指定子进程已经终止。
```

父进程阻塞在 `join()` 时，不会顺便替你调用 `parent_conn.recv()`，也不会主动消费通信通道中的 HTML。它只是等待子进程退出。

```text
父进程继续执行的条件 = 子进程已经终止
```

#### 4.2 `recv()` 等待并接收完整消息

```python
html = parent_conn.recv()
```

它的语义可以概括为：

```text
从 Connection 接收下一条完整消息；如果消息还没有准备好，就等待。
```

这里的等待同时承担“消费数据”的角色。子进程写入时，父进程的读取会释放底层传输缓冲区空间，使后续字节能够继续流动。

```text
父进程继续执行的条件 = 收到一条完整消息
```

因此区别不是“阻塞”与“不阻塞”，而是：

```text
join()：阻塞等待进程终止，不消费结果通道。
recv()：阻塞等待消息，并在消息到达时持续消费结果通道。
```


### 5. 错误顺序为什么产生死锁

错误顺序：

```python
process.start()
process.join()
html = parent_conn.recv()
```

假设：

```text
HTML 序列化后大小：500 KiB
底层可用发送缓冲区：约 64 KiB
```

#### 5.1 阶段一：父进程启动子进程

```text
父进程                              子进程
process.start()  ─────────────────>  开始执行 target
```

#### 5.2 阶段二：父进程进入 `join()`

```text
父进程                              子进程
process.join()                      generate_html()
等待子进程退出                      生成 500 KiB HTML
```

此时父进程没有执行 `recv()`。

#### 5.3 阶段三：子进程调用 `send()`

```text
父进程                              子进程
仍阻塞在 join()                     conn.send(html)
```

`multiprocessing.Connection.send()` 需要先序列化 Python 对象，再把包含消息边界信息的数据写入底层通信通道。它可以发送超过底层缓冲区容量的对象，但前提是接收端同步地读取数据。

底层传输刚开始时：

<details>
<summary>展开时序/结构示意（13 行）</summary>

```text
子进程内存中的序列化消息
┌──────────────────────────────────────────┐
│ 500 KiB                                  │
└──────────────────────────────────────────┘
             │ 写入
             ▼
内核通信缓冲区
┌──────────────────┐
│ 约 64 KiB，已满   │
└──────────────────┘
             │ 等待父进程读取
             ▼
父进程当前却停在 join()
```

</details>

Linux 对满管道执行阻塞式写入时，写入进程会等待，直到读取端取走数据、释放足够空间。Linux 管道是容量有限的内核缓冲区，并不是无限消息仓库。[Linux `pipe(7)`](https://www.man7.org/linux/man-pages/man7/pipe.7.html)

#### 5.4 阶段四：依赖形成闭环

此时：

<details>
<summary>展开时序/结构示意（9 行）</summary>

```text
父进程等待子进程退出
            ↓
子进程退出前必须完成 send()
            ↓
send() 要完成，需要父进程读取数据
            ↓
父进程却规定：等 join() 完成以后才执行 recv()
            ↓
回到最开始：join() 又在等子进程退出
```

</details>

用依赖图表示：

<details>
<summary>展开时序/结构示意（10 行）</summary>

```text
recv()
   ↑
必须先越过 join()
   ↑
join() 必须等 child exit
   ↑
child exit 必须等 send() 返回
   ↑
send() 必须等 recv() 排空缓冲区
   └──────────────────────────────┘
```

</details>

这是典型的循环等待：

```text
recv → join → child exit → send → recv
```

所以程序不是“HTML 处理很慢”，而是没有任何一方能够独立打破依赖环。


### 6. 为什么改成先 `recv()`、再 `join()` 就能完成

正确顺序：

```python
process.start()
html = parent_conn.recv()
process.join()
```

其时序是：

<details>
<summary>展开时序/结构示意（14 行）</summary>

```text
父进程                              子进程
process.start()                     开始执行
       ↓                                 ↓
parent_conn.recv()                  generate_html()
等待消息并准备读取                         ↓
       │                           conn.send(html)
       │<────── 分段流动的数据 ────────────│
持续读取和释放缓冲空间                    持续写入
       │                                 ↓
获得完整 HTML                       send() 返回
       ↓                                 ↓
process.join()                     子进程退出
       ↓
join() 返回
```

</details>

现在没有循环依赖：

```text
子进程生成 HTML
        ↓
子进程 send()  ←→  父进程 recv()
        ↓
子进程退出
        ↓
父进程 join() 返回
```

父进程仍然负责启动子进程，但 `start()` 后二者可以并发执行。把 `recv()` 放在前面，并不是让父进程“抢在子进程之前取得结果”，而是让父进程进入接收状态：没有数据时它等待；数据到达后它消费；缓冲区被释放后，子进程继续写入。传输双方因此可以共同推进。


### 7. 为什么只调用一次 `recv()`，却能接收远大于 64 KiB 的 HTML

应用代码只有一句：

```python
html = parent_conn.recv()
```

但高层 API 的一次调用不等于底层只执行一次系统读取。`recv()` 知道消息的封装格式，可以读取消息长度并持续读取，直到完整消息到齐，然后反序列化并返回 Python 对象。

可将内部逻辑概念化为：

<details>
<summary>展开 Python 代码（10 行）</summary>

```python
def conceptual_recv(connection):
    message_length = read_message_header(connection)
    chunks = []
    received = 0

    while received < message_length:
        chunk = low_level_read(connection)
        chunks.append(chunk)
        received += len(chunk)

    serialized_object = b"".join(chunks)
    return deserialize(serialized_object)
```

</details>

这不是 CPython 源码的逐行复制，而是为了说明高层语义。一次 `recv()` 内部可能完成许多次底层读取：

```text
第一次读取：一部分字节
第二次读取：另一部分字节
第三次读取：继续读取
……
最后一次读取：收齐该消息
```

因此：

```text
64 KiB 缓冲容量
≠ 一条消息最大只能是 64 KiB
```

准确关系是：

```text
缓冲容量限制的是某一时刻“已经写入但尚未读取”的在途数据量，
不限制通过持续写入和持续读取传输的总数据量。
```

可以把缓冲区理解成传送带，而不是仓库：

```text
子进程的 500 KiB HTML
         ↓
 [约 64 KiB 的内核中转缓冲区]
         ↓
父进程最终还原出的 500 KiB HTML
```

只要消费者持续取走数据，小缓冲区也能传输大对象。


### 8. 为什么 `recv()` 的阻塞正常，而 `join()` 的阻塞会形成死锁

#### 正常等待

```text
父进程：recv() 等待消息
子进程：仍可继续 generate_html()，完成后调用 send()
```

父进程等待的是子进程可以独立产生的事件。子进程不需要父进程先退出 `recv()`，反而正需要父进程处于接收状态。

#### 死锁等待

```text
父进程：join() 等待子进程退出
子进程：send() 等待父进程读取
父进程：只有 join() 返回后才会读取
```

判断等待是否危险，可以问一句：

> 我正在等待的事件，对方能否在不等待我的情况下独立完成？

如果能完成，通常是正常阻塞；如果对方完成该事件又依赖你先做某件尚未执行的事，就可能形成死锁。

</details>

### 9. 推荐实现一：`Pipe` 先接收结果，再回收子进程

下面是适合“父进程必须直接获得 HTML 字符串”的完整基础实现：

<details>
<summary>展开 Python 代码（37 行）</summary>

```python
import multiprocessing
import traceback


def generate_html() -> str:
    return "<html><body>" + ("x" * 500_000) + "</body></html>"


def child_process(conn) -> None:
    try:
        html = generate_html()
        conn.send({
            "ok": True,
            "html": html,
        })
    except Exception:
        # Return child-side error details to the parent.
        conn.send({
            "ok": False,
            "traceback": traceback.format_exc(),
        })
    finally:
        conn.close()


if __name__ == "__main__":
    parent_conn, child_conn = multiprocessing.Pipe(duplex=False)

    process = multiprocessing.Process(
        target=child_process,
        args=(child_conn,),
    )

    process.start()

    # The parent must close its duplicate of the child's write endpoint.
    child_conn.close()

    # Receive first so the result channel is drained while the child sends.
    result = parent_conn.recv()

    # Join only after the child has finished sending the result.
    process.join()
    parent_conn.close()

    if not result["ok"]:
        raise RuntimeError(result["traceback"])

    html = result["html"]
    print(f"Received HTML length: {len(html)}")
```

</details>

#### 为什么关闭父进程持有的 `child_conn`

创建 pipe 后，父进程一开始持有两个端点；启动子进程时，子进程获得它要使用的端点。父进程应关闭自己不使用的写端副本。否则，即使真正的子进程已经关闭写端，系统仍可能认为还有写端处于打开状态，从而影响 EOF 检测和资源释放。Linux 文档明确建议在 `fork()`/pipe 场景关闭不必要的重复文件描述符。[Linux `pipe(7)`](https://www.man7.org/linux/man-pages/man7/pipe.7.html)


### 10. 推荐实现二：加入超时、异常传递和退出状态检查

仅交换 `recv()` 和 `join()` 能解决大型结果的循环等待，但生产代码还要考虑：

- 子进程在发送前崩溃；
- 子进程生成 HTML 无限等待；
- 子进程被信号终止，来不及发送错误对象；
- 父进程永久卡在 `recv()`；
- 收到结果后，子进程清理阶段异常或不退出。

更健壮的版本：

<details>
<summary>展开 Python 代码（58 行）</summary>

```python
import multiprocessing
import traceback


def generate_html() -> str:
    return "<html><body>" + ("x" * 500_000) + "</body></html>"


def child_process(conn) -> None:
    try:
        html = generate_html()
        conn.send({
            "ok": True,
            "html": html,
        })
    except Exception:
        try:
            conn.send({
                "ok": False,
                "traceback": traceback.format_exc(),
            })
        except Exception:
            # The communication channel itself may already be broken.
            pass
    finally:
        conn.close()


def run_child(timeout_seconds: float = 300.0) -> str:
    parent_conn, child_conn = multiprocessing.Pipe(duplex=False)
    process = multiprocessing.Process(
        target=child_process,
        args=(child_conn,),
    )

    process.start()
    child_conn.close()

    try:
        if not parent_conn.poll(timeout_seconds):
            process.terminate()
            process.join(timeout=10)
            raise TimeoutError(
                f"Child process did not return a result within "
                f"{timeout_seconds} seconds"
            )

        result = parent_conn.recv()

        process.join(timeout=10)
        if process.is_alive():
            process.terminate()
            process.join(timeout=10)
            raise TimeoutError(
                "Child returned a result but did not terminate cleanly"
            )

        if process.exitcode != 0 and result.get("ok"):
            raise RuntimeError(
                f"Child returned data but exited with code {process.exitcode}"
            )

        if not result.get("ok"):
            raise RuntimeError(result.get("traceback", "Unknown child error"))

        return result["html"]
    finally:
        parent_conn.close()


if __name__ == "__main__":
    html = run_child()
    print(f"Received HTML length: {len(html)}")
```

</details>

#### `poll(timeout)` 的准确作用

```python
parent_conn.poll(300)
```

用于等待“当前 connection 是否有可读取消息”，并提供超时控制。它避免子进程在发送前异常退出时，父进程无限等待 `recv()`。

它不是大型消息死锁的根本修复。根本修复仍然是：

```text
发送大结果期间，父进程必须能够读取结果通道；
不能先无限等待子进程终止。
```

#### 谨慎使用 `terminate()`

强制终止进程可能使正在使用的 pipe、queue、lock 或其他共享资源处于损坏或不可用状态。Python 官方文档对此有明确警告。因此应优先设计正常退出与协作式取消，`terminate()` 主要作为超时后的最后手段。[Python `multiprocessing` 文档](https://docs.python.org/3/library/multiprocessing.html)


### 11. 使用 `multiprocessing.Queue` 时为什么也会发生类似问题

错误模式：

<details>
<summary>展开 Python 代码（9 行）</summary>

```python
from multiprocessing import Process, Queue


def child(queue):
    html = generate_html()
    queue.put(html)


queue = Queue()
process = Process(target=child, args=(queue,))
process.start()
process.join()
html = queue.get()
```

</details>

`multiprocessing.Queue` 会序列化对象，并使用后台 feeder thread 把缓冲数据送入底层 pipe。`put()` 返回不一定意味着所有字节已经被父进程消费。发送进程退出时，可能还需等待 feeder thread 把缓冲数据送完。如果父进程先 `join()` 而不 `get()`，大型对象可能堵住底层通道，从而产生与 Pipe 类似的等待环。

Python 官方文档专门给出了这种会死锁的示例，并说明：使用 Queue 时，需要确保已放入 Queue 的项目最终在发送进程被 join 之前被移除，否则不能保证发送进程能够终止。[Python `multiprocessing` 文档](https://docs.python.org/3/library/multiprocessing.html)

正确基本顺序：

<details>
<summary>展开 Python 代码（30 行）</summary>

```python
from multiprocessing import Process, Queue
from queue import Empty
import traceback


def child(result_queue):
    try:
        result_queue.put({
            "ok": True,
            "html": generate_html(),
        })
    except Exception:
        result_queue.put({
            "ok": False,
            "traceback": traceback.format_exc(),
        })


if __name__ == "__main__":
    result_queue = Queue()
    process = Process(target=child, args=(result_queue,))
    process.start()

    try:
        result = result_queue.get(timeout=300)
    except Empty:
        process.terminate()
        process.join()
        raise TimeoutError("Child process did not return a result")

    process.join()
    result_queue.close()
    result_queue.join_thread()

    if not result["ok"]:
        raise RuntimeError(result["traceback"])

    html = result["html"]
```

</details>

不要依靠 `Queue.empty()` 判断下一次 `get()` 是否安全。并发环境下，状态在检查和使用之间可能发生变化，且 Queue 的 feeder 行为也会造成短暂可见性延迟。应直接使用带超时的 `get()` 或明确的消息协议。


### 12. 与你的 HTML 场景直接对应的最终建议

如果业务要求是：

```text
子进程负责生成 HTML；
父进程必须拿到完整 HTML 字符串；
然后父进程继续返回响应或做后处理。
```

采用以下顺序：

```python
process.start()
result = parent_conn.recv()
process.join()
```

并补齐：

1. 使用 `Pipe(duplex=False)` 表达单向结果通道；
2. 父进程关闭自己持有的子写端副本；
3. 子进程通过统一 envelope 返回成功或错误；
4. 使用 `poll(timeout)`，避免子进程未发送时父进程永久等待；
5. 收到结果后再 `join(timeout)`；
6. 检查 `exitcode`；
7. 最终关闭 connection；
8. 不以调大 pipe capacity 代替正确控制流；
9. 如果 HTML 规模或频率继续增长，评估文件、对象存储、流式分块或共享内存方案。

最重要的因果关系是：

```text
start() 只负责启动并发执行；
recv() 负责参与结果传输并排空通信通道；
join() 负责在通信完成后等待和回收子进程。
```

因此正确理解不是“神奇地交换两行代码”，而是：

```text
先让通信协议完成，再执行进程生命周期回收。
```

错误顺序表达的是：

```text
先等子进程退出，再读取它的结果；
但子进程必须先把结果交给父进程，才能退出。
```

正确顺序表达的是：

```text
父进程在子进程发送期间持续接收；
结果传输结束后，子进程可以退出；
父进程最后 join，确认资源被回收。
```


<details>
<summary>最小复现实验：错误顺序与正确顺序对照（第 13 节）</summary>

### 13. 最小复现实验

下面的实验可以直观看出顺序差异。请在测试环境运行，不要让死锁版本无限挂住。

#### 可能死锁的版本

<details>
<summary>展开 Python 代码（17 行）</summary>

```python
import multiprocessing


def child(conn):
    payload = "x" * 1_000_000
    print("child: before send", flush=True)
    conn.send(payload)
    print("child: after send", flush=True)
    conn.close()


if __name__ == "__main__":
    parent_conn, child_conn = multiprocessing.Pipe(duplex=False)
    process = multiprocessing.Process(target=child, args=(child_conn,))

    process.start()
    child_conn.close()

    print("parent: before join", flush=True)
    process.join()
    print("parent: after join", flush=True)

    payload = parent_conn.recv()
    print("parent: received", len(payload), flush=True)
```

</details>

典型观察：

```text
parent: before join
child: before send
```

之后双方不再推进。

#### 正常版本

<details>
<summary>展开 Python 代码（17 行）</summary>

```python
import multiprocessing


def child(conn):
    payload = "x" * 1_000_000
    print("child: before send", flush=True)
    conn.send(payload)
    print("child: after send", flush=True)
    conn.close()


if __name__ == "__main__":
    parent_conn, child_conn = multiprocessing.Pipe(duplex=False)
    process = multiprocessing.Process(target=child, args=(child_conn,))

    process.start()
    child_conn.close()

    print("parent: before recv", flush=True)
    payload = parent_conn.recv()
    print("parent: received", len(payload), flush=True)

    process.join()
    print("parent: joined", flush=True)
```

</details>

典型观察：

```text
parent: before recv
child: before send
child: after send
parent: received 1000000
parent: joined
```

这里的一百万字符远大于常见的 64 KiB 缓冲量，但能够传输成功，因为父进程在子进程发送期间同步读取。



---

</details>
