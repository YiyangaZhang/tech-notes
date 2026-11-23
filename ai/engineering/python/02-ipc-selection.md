# 02 IPC 选型、容量与诊断

> 原稿最早可见 Git 记录：2026-10-14；原稿最近 Git 修改：2026-10-15。公开版整理：2026-11-17。以上 Git 日期用于追溯，不等同于实际首次写作日期。
> 本文为通用知识整理，历史产品能力及版本细节需结合当前文档复核。

章节编号沿用原稿，便于在系列中回溯；长代码与推导可折叠阅读。

## IPC 选择与诊断

下列内容用于按数据规模、通信拓扑和持久化要求选择 IPC；Linux pipe 容量的历史背景与背压理论见后续折叠章节。

<details>
<summary>IPC 基础与方案选择：pipe、socket、队列、共享内存和文件（第 24–37 节）</summary>

### 24. 系统内存变大为什么通常无助于这个问题

机器总内存与单个 IPC 缓冲区容量不是同一层资源：

```text
系统 RAM：整栋仓库，例如 128 GiB
单个 pipe/socket buffer：一个受控中转货架，例如几十 KiB或更多
```

即使系统有 128 GiB RAM，内核也不会默认允许每条 pipe 无限增长。原因包括：

- 系统可能同时存在大量 pipe 和 socket；
- 生产者可能远快于消费者；
- 无限缓存会把速度不匹配转化为不可控内存增长；
- 有限缓冲区可以对发送方实施背压。

因此，这里的阻塞通常不是“系统没有空闲 RAM”，而是“这条通信通道的有限缓冲区已经满，而消费者没有读取”。增加机器 RAM 不会改变错误控制流：

```text
join → 等子进程退出 → send 等 recv → recv 又在 join 之后
```

即使缓冲区因为配置变化而更大，只要 HTML 最终超过它，问题仍会重现。


### 25. 可以调大 Linux pipe buffer 吗？为什么不应作为根治方案

Linux 提供 `F_GETPIPE_SZ` 和 `F_SETPIPE_SZ` 来读取或调整 pipe capacity。普通用户可设置的范围受页面大小、`/proc/sys/fs/pipe-max-size` 及用户 pipe page 限制影响；请求值还可能被内核向方便分配的页面倍数取整。[Linux `F_GETPIPE_SZ` / `F_SETPIPE_SZ`](https://www.man7.org/linux/man-pages/man2/F_SETPIPE_SZ.2const.html)

查看系统设置：

```bash
cat /proc/sys/fs/pipe-max-size
cat /proc/sys/fs/pipe-user-pages-soft
cat /proc/sys/fs/pipe-user-pages-hard
getconf PAGESIZE
```

C 层概念示例：

```c
int current_size = fcntl(fd, F_GETPIPE_SZ);
int actual_size = fcntl(fd, F_SETPIPE_SZ, 1024 * 1024);
```

但这不是推荐的根治方式：

```text
当前 HTML 500 KiB，缓冲区改成 1 MiB
→ 当前测试可能不再阻塞

未来 HTML 2 MiB
→ 同一问题再次出现
```

并且 `multiprocessing.Pipe()` 的具体底层机制具有平台差异，双工连接也不应简单假定为“一个可以直接用 `F_SETPIPE_SZ` 调整的普通匿名 pipe”。正确设计应允许发送端和接收端并发推进，而不是假设一条完整消息一定可以一次性塞进内核缓冲区。

调大缓冲区更适合性能调优，例如减少系统调用频率、吸收短暂流量尖峰；不适合修复循环等待。


### 26. Python `Pipe`、Linux pipe 与 socketpair 的关系要谨慎区分

#### Linux 匿名 pipe

`pipe()` 创建单向字节流，返回读端和写端两个文件描述符。写入的数据由内核缓存，直到读取端取走。[Linux `pipe(2)`](https://www.man7.org/linux/man-pages/man2/pipe.2.html)

```text
writer ── bytes ──> kernel pipe buffer ── bytes ──> reader
```

#### `socketpair()`

`socketpair()` 创建一对已连接的 socket，两个端点地位对等，适合同机双向通信。Linux 上常见域是 `AF_UNIX`。[Linux `socketpair(2)`](https://www.man7.org/linux/man-pages/man2/socketpair.2.html)

```text
endpoint A <──────── full duplex ────────> endpoint B
```

#### Python `multiprocessing.Pipe`

Python 的 `multiprocessing.Pipe(duplex=True)` 返回两个 `Connection` 对象。它提供的是带消息边界的高层对象传输接口，`send()` 会序列化对象，`recv()` 会还原对象。Python 文档将 Pipe 和 Queue 作为进程消息传递方式，但其底层实现会依平台和 duplex 模式变化。[Python `multiprocessing` 文档](https://docs.python.org/3/library/multiprocessing.html)

因此不要把三个概念完全等同：

```text
Linux pipe：内核字节流机制
socketpair：成对连接的 socket
multiprocessing.Pipe：Python Connection 抽象及对象消息协议
```

它们都可能存在有限缓冲区、阻塞写入和背压，但细节并不完全相同。


### 27. 为什么进程间需要专门通信

每个进程通常有独立虚拟地址空间：

```text
进程 A 的变量 html
≠
进程 B 地址空间中的变量 html
```

子进程生成 Python 字符串后，父进程不能直接读取子进程里的那个 Python 对象。必须通过以下方式之一交换信息：

1. 内核转发字节或消息；
2. 双方映射同一片共享内存；
3. 子进程落盘，父进程读取文件；
4. 使用更高层 RPC、数据库或对象存储。

选择 IPC 时，通常考虑：

- 单向还是双向；
- 一对一还是多生产者/多消费者；
- 字节流还是有消息边界；
- 数据大小；
- 是否跨主机；
- 是否需要持久化；
- 吞吐量与延迟；
- 失败恢复和超时；
- 权限、安全与访问控制；
- 实现复杂度。


### 28. 匿名管道 Anonymous Pipe

特点：

- 通常用于有亲缘关系的进程，例如父子进程；
- 原生 Linux pipe 是单向字节流；
- 没有天然消息边界；
- 内核维护有限容量的缓冲区；
- 空管道的阻塞读取会等待数据；
- 满管道的阻塞写入会等待空间；
- 写端全部关闭后，读取端读到 EOF；
- 读端全部关闭后，写入会触发 `SIGPIPE`，或在忽略该信号时返回 `EPIPE`。[Linux `pipe(7)`](https://www.man7.org/linux/man-pages/man7/pipe.7.html)

适合：

```text
parent → child 任务参数
child → parent 小型或流式结果
command1 | command2 shell pipeline
```

不适合：

- 需要复杂多方路由；
- 需要跨主机；
- 需要消息持久化；
- 双方都采用“先发送全部大数据，再接收”的协议。

#### `PIPE_BUF` 与 pipe capacity 不是一回事

- **pipe capacity**：缓冲区总共能容纳多少尚未读取的数据；满后写入可能阻塞。
- **`PIPE_BUF`**：与多个写入者并发写入时，某个写操作是否保证不与其他写入交错相关的原子性界限。

不要把“原子写入上限”“当前容量”“单条消息大小上限”混为一谈。原生 pipe 本身只是字节流，没有消息概念。[Linux `pipe(7)`](https://www.man7.org/linux/man-pages/man7/pipe.7.html)


### 29. Named Pipe / FIFO

FIFO 与匿名 pipe 的 I/O 语义基本相同，但它在文件系统中有路径：

```bash
mkfifo /tmp/my_fifo
```

不同进程无需具有父子关系，只要有权限访问该路径即可打开 FIFO。

适合：

- 简单本机进程协作；
- shell 工具之间连接；
- 临时数据流。

注意：

- FIFO 路径只是访问入口，数据本身并不是普通文件内容；
- 打开 FIFO 也可能因对端尚未打开而阻塞；
- 仍然需要处理容量、EOF、`SIGPIPE`、权限和端点关闭。


### 30. Unix Domain Socket 与 `socketpair`

Unix Domain Socket（UDS）用于同一主机进程通信，可提供：

- 字节流 `SOCK_STREAM`；
- 数据报 `SOCK_DGRAM`；
- 双向通信；
- 文件系统路径或抽象命名空间寻址；
- 较自然的 server/client 结构；
- 在 Linux 上传递文件描述符等高级能力。

`socketpair()` 则直接创建一对已连接 socket，常用于父子进程或本地紧密协作进程。[Linux `socketpair(2)`](https://www.man7.org/linux/man-pages/man2/socketpair.2.html)

需要注意：socket 也有发送和接收缓冲区。对端不读时，阻塞式 `send()` 最终也可能阻塞。因此换成 socket 并不会自动消除错误的“双方先发完再收”协议。

适合：

- 本机双向客户端/服务端；
- 需要 `select/poll/epoll` 管理多个连接；
- 需要比 pipe 更灵活的通信拓扑；
- 希望未来较容易迁移到 socket 风格协议。


### 31. TCP/IP Socket

TCP socket 可跨主机通信，提供可靠、有序的字节流，但同样没有应用层消息边界。应用需要定义 framing，例如：

```text
[length header][payload]
```

或使用 JSON Lines、HTTP、gRPC 等高层协议。

TCP 也有流量控制和发送缓冲。`send()` 成功通常只说明数据已被本机协议栈接受，不代表远端应用已经处理。对端不读取、网络拥塞或窗口耗尽时，阻塞发送仍可能停住。

适合：

- 跨主机；
- 服务化 API；
- 需要成熟网络协议生态。

不应仅为了父子进程传一个 HTML 就引入 TCP，除非系统本身已经服务化。


### 32. 消息队列

消息队列强调消息边界，而不是裸字节流。Linux/POSIX 层面有消息队列，Python `multiprocessing.Queue` 则是更高层的 Python 对象队列。

优势：

- 多生产者、多消费者更自然；
- 每条消息边界清晰；
- 解耦生产与消费；
- 容易表达任务队列或结果队列。

限制：

- 仍然必须有容量或资源限制；
- 大对象需要序列化和复制；
- 队列满或底层传输堵塞时仍会背压；
- 进程退出、feeder thread、队列清理顺序可能带来额外复杂度。

对于“一个子进程返回一个 HTML”，Pipe 更直接；对于“多个 worker 返回多个任务结果”，Queue 通常更自然。


### 33. 共享内存 Shared Memory

共享内存让多个进程映射同一片物理内存，避免通过 pipe 复制完整数据：

```text
进程 A 虚拟地址 ─┐
                  ├── 同一片共享物理内存
进程 B 虚拟地址 ─┘
```

优势：

- 大型二进制数据吞吐高；
- 避免多次序列化和复制；
- 适合 NumPy 数组、图像、大型固定结构数据。

代价：

- 必须自行设计数据布局；
- 必须处理同步、生命周期和清理；
- 锁、信号量或状态标志设计错误仍会死锁或产生竞态；
- Python 字符串和任意对象不能简单地“共享一个对象引用”。

对于 HTML 字符串，临时文件或正确使用 Pipe 往往比共享内存简单。只有当结果非常大、传输频繁且复制成本已经被测量为瓶颈时，再考虑共享内存。


### 34. Memory-mapped File

`mmap` 将文件映射到进程地址空间。多个进程可以映射同一文件，兼具共享访问和文件后备的特性。

适合：

- 大型结果；
- 需要随机访问；
- 希望结果可在进程退出后保留；
- 需要把大 payload 与小型控制消息分开。

一种实用模式：

```text
子进程写 mmap/临时文件
        ↓
Pipe 仅发送路径、长度、校验值和状态
        ↓
父进程按需读取或映射
```


### 35. Signal、Event、Semaphore 和 Lock

这些机制主要传递“状态或同步关系”，而不是大型数据：

- **signal**：异步通知，例如终止、重载配置；
- **Event**：表示某个条件是否已发生；
- **Semaphore**：控制有限资源的并发访问数量；
- **Lock/Mutex**：保证临界区互斥；
- **Condition**：等待某个由共享状态表达的条件；
- **Barrier**：等待一组参与者都到达同步点。

它们常与共享内存或队列配合，而不是代替数据通道。例如：

```text
共享内存存 HTML bytes
Event 表示“写入完成”
```

但同步原语使用不当也会形成死锁，例如重复获取同一非重入锁、不同进程以相反顺序获取多个锁、某个异常路径遗漏释放。


### 36. 文件作为 IPC

对大型 HTML，文件其实是非常合理的 IPC 介质：

<details>
<summary>展开时序/结构示意（9 行）</summary>

```text
子进程生成 HTML
        ↓
原子写入临时文件
        ↓
fsync/close（按持久化要求选择）
        ↓
重命名到最终路径
        ↓
通过 Pipe 返回路径、大小、hash、状态
```

</details>

优势：

- 不受小型 pipe buffer 容量约束；
- 可以保留结果用于下载和审计；
- 父进程不必同时在内存里持有额外副本；
- 子进程退出后结果仍存在；
- 特别适合本来就要提供 HTML 文件或 URL 的 Web 服务。

代价：

- 文件系统 I/O；
- 生命周期和垃圾清理；
- 路径安全与权限；
- 部分写入可见性，需要临时文件加原子 rename；
- 多实例或容器环境中，本地路径未必共享。

如果业务硬性要求父进程直接得到字符串，那么应使用先 `recv()` 后 `join()`；如果真正需求只是让父进程把报告返回给客户端，路径/对象存储 key/streaming response 可能更合理。


### 37. 如何选择 IPC 方式

#### 场景 A：一个子进程返回一个中小型 HTML 字符串

选择：

```text
multiprocessing.Pipe(duplex=False)
```

要求：

- 父进程先 `recv()`，再 `join()`；
- 有超时和错误消息协议；
- 关闭无用端点。

#### 场景 B：多个 worker 返回多个结果

选择：

```text
multiprocessing.Queue
```

要求：

- 消费结果后再 join 生产者；
- 使用超时；
- 设计结束标记或任务计数；
- 不依赖 `empty()` 完成并发正确性判断。

#### 场景 C：HTML 很大，最终本来就要保存

选择：

```text
文件 / 对象存储 + Pipe 返回元数据
```

#### 场景 D：高频传输大型数组

选择：

```text
shared_memory / mmap + 同步协议
```

#### 场景 E：本机长期服务与多个客户端

选择：

```text
Unix Domain Socket
```

#### 场景 F：跨主机服务

选择：

```text
TCP/HTTP/gRPC/消息中间件
```


</details>

<details>
<summary>大结果传输、分块协议与诊断方法（第 38–40 节）</summary>

### 38. 大型 HTML 传输中的额外成本

当子进程执行：

```python
conn.send(html)
```

可能涉及：

<details>
<summary>展开时序/结构示意（9 行）</summary>

```text
子进程中的 Python str
        ↓ 序列化
序列化后的 bytes
        ↓ 复制/写入
内核通信缓冲区
        ↓ 读取/复制
父进程用户空间
        ↓ 反序列化
父进程中的新 Python str
```

</details>

这意味着：

- 父子进程各自可能持有 HTML 副本；
- 序列化过程可能产生临时内存；
- 大对象频繁传输会增加 CPU、内存带宽和内核切换成本；
- 系统总内存较大只会让复制更不容易 OOM，不会修复通信时序。

如果 HTML 只是一次性几十或几百 KiB，正确使用 Pipe 通常足够。不要在没有性能测量前过度复杂化。如果 HTML 达到几十 MiB、返回频率高，或父进程只需要转发到文件/HTTP，则应重新评估是否必须把完整字符串跨进程复制。


### 39. 分块传输：什么时候需要，以及怎么设计

`Connection.send()` 已能传大对象，不必仅因为超过 64 KiB 就手动分块。手动分块适用于：

- 希望边生成边消费，而不是先构造完整 HTML；
- 希望限制峰值内存；
- 希望报告进度；
- 希望取消传输；
- 单个对象过大，序列化峰值不可接受。

消息协议示例：

```text
START:  {request_id, content_type, optional_total_size}
CHUNK:  {request_id, sequence, bytes}
CHUNK:  {request_id, sequence, bytes}
END:    {request_id, chunk_count, checksum}
ERROR:  {request_id, error_type, message}
```

概念代码：

<details>
<summary>展开 Python 代码（13 行）</summary>

```python
CHUNK_SIZE = 64 * 1024


def child(conn):
    try:
        conn.send(("START", None))

        for html_chunk in generate_html_chunks():
            encoded = html_chunk.encode("utf-8")
            for offset in range(0, len(encoded), CHUNK_SIZE):
                conn.send_bytes(encoded[offset:offset + CHUNK_SIZE])

        conn.send(("END", None))
    except Exception as exc:
        conn.send(("ERROR", str(exc)))
    finally:
        conn.close()
```

</details>

实际实现时不要混乱使用 `send()` 和 `send_bytes()` 而没有明确 framing，因为接收端必须知道下一段按哪种协议解释。以上仅用于展示设计方向；生产实现应统一消息格式，处理错误、序号、超时、取消和完整性校验。


### 40. 诊断：如何确认真的是管道/IPC 阻塞

不要只根据“约 64 KiB 后卡住”就下结论。应验证父子进程各自在等待什么。

#### 40.1 先在关键位置记录日志

<details>
<summary>展开 Python 代码（9 行）</summary>

```python
# Parent
logger.info("starting child")
process.start()
logger.info("child started: pid=%s", process.pid)
logger.info("waiting for child result")
result = parent_conn.recv()
logger.info("result received")
process.join()
logger.info("child joined: exitcode=%s", process.exitcode)
```

</details>

```python
# Child
logger.info("generating HTML")
html = generate_html()
logger.info("HTML generated: chars=%s", len(html))
logger.info("sending HTML")
conn.send(html)
logger.info("HTML sent")
```

如果最后日志是：

```text
Parent: waiting in join
Child: sending HTML
```

而没有：

```text
Child: HTML sent
```

就高度符合发送阻塞与 join/recv 顺序问题。

#### 40.2 使用 `strace`

附加到父进程和子进程：

```bash
strace -ff -tt -T -p <parent_pid>
strace -ff -tt -T -p <child_pid>
```

或者从启动阶段跟踪：

```bash
strace -ff -tt -T -o /tmp/app.strace python your_app.py
```

重点观察：

```text
父进程是否停在 wait4()/waitid() 一类等待子进程调用；
子进程是否停在 write()/sendmsg()/sendto() 一类发送调用；
父进程是否根本没有 read()/recvmsg()。
```

不同 Python 版本和底层实现对应的系统调用可能不同，不要机械地只搜索 `write`。

#### 40.3 查看进程状态

```bash
ps -o pid,ppid,stat,wchan:32,cmd -p <parent_pid>,<child_pid>
```

`wchan` 可以提示内核等待位置，但符号展示受权限和内核配置影响。

#### 40.4 查看文件描述符

```bash
ls -l /proc/<pid>/fd
lsof -p <pid>
```

确认 pipe/socket 端点是否仍被不该持有的进程打开。

#### 40.5 检查 Python 栈

可使用：

- `faulthandler` 定时输出 Python 栈；
- 调试器；
- `py-spy dump --pid <pid>`，如果环境允许；
- 在 `join()`、`recv()`、`send()` 前后插入明确日志。

最有价值的证据不是“程序没响应”，而是：

```text
父进程 Python 栈：Process.join()
子进程 Python 栈：Connection.send()
系统调用层：父在 wait，子在 write/send
```

</details>

### 41. 常见误区

#### 误区 1：`send()` 最多发送 64 KiB

错误。缓冲区容量不是消息总大小上限。接收端持续读取时，可以传输远大于缓冲区的数据。

#### 误区 2：机器 RAM 越大，pipe 就自动越大

错误。总 RAM 与单条 IPC 通道的容量策略不同。

#### 误区 3：`join()` 是标准清理动作，所以应该紧跟在 `start()` 后

不一定。`join()` 的位置属于通信协议的一部分。如果子进程退出依赖父进程先消费结果，必须先消费再 join。

#### 误区 4：`recv()` 也会阻塞，所以和 `join()` 没区别

错误。`recv()` 在数据到达后会消费通道并释放空间；`join()` 只等进程终止。

#### 误区 5：把缓冲区调到 1 MiB 就解决了

只会提高触发阈值，不能消除循环依赖。

#### 误区 6：换成 Queue 就一定安全

错误。Queue 同样涉及有限底层通道、序列化和清理顺序；大型对象仍可能在“先 join 后 get”的顺序中造成死锁。

#### 误区 7：子进程 `put()` 或 `send()` 一调用，数据就已经在父进程里

错误。调用可能阻塞，也可能只完成部分内部阶段；Queue 还可能涉及 feeder thread。

#### 误区 8：只要加无限重试就可以

错误。循环等待不是暂时失败，无限重试可能只是无限消耗时间或 CPU。


<details>
<summary>Linux pipe 容量深入：64 KiB、PIPE_BUF 与系统限制（第 42–51 节）</summary>

### 42. 先纠正一个表述：Linux 并没有把所有管道永远“定死”为 64 KiB

更准确的说法是：

```text
在许多采用 4 KiB page 的 Linux 环境中，传统默认 pipe capacity 常见为 16 pages：
16 × 4096 B = 65,536 B = 64 KiB。
```

但它不是跨版本、跨架构、跨系统和跨 IPC 类型的固定常量：

- Linux 2.6.11 之前，pipe capacity 曾经通常只有一个 page；
- 从 Linux 2.6.11 起，典型容量变成 16 pages；
- 从 Linux 2.6.35 起，可以通过 `F_GETPIPE_SZ` 和 `F_SETPIPE_SZ` 查询或调整单个 pipe；
- 从 Linux 4.5 起，默认容量还可能因用户级 pipe page soft limit 而低于 16 pages；
- 从 Linux 4.9 起，`/proc/sys/fs/pipe-max-size` 也可作为新建 pipe/FIFO 默认容量的上限；
- 实际 page size 可能不是 4 KiB；
- Python `multiprocessing.Pipe(duplex=True)` 的底层实现也不应简单等同于一个普通匿名 pipe。

因此工程上不要写死假设：

```python
PIPE_CAPACITY = 65536
```

而应把容量视为平台和运行时属性。

参考：Linux `pipe(7)` 说明 pipe 容量有限、满时阻塞，并记录了不同内核版本中的容量变化；`F_GETPIPE_SZ/F_SETPIPE_SZ` 用于查询和设置单个 pipe 的容量。


### 43. 为什么是“16 个 page”，而不是随便选择 65,536 这个数字

Linux 内核管理内存的基本单位之一是 page。pipe 的数据缓冲也基于 page 和环形缓冲槽位组织，而不是专门为每条 pipe 分配一个任意字节数的大连续内存块。

在常见 4 KiB page 系统中：

```text
单个 page：4 KiB
默认槽位：16 pages
总容量：16 × 4 KiB = 64 KiB
```

所以从实现视角看，关键数字更接近：

```text
16 pages
```

64 KiB 只是它在 4 KiB page 机器上的结果。

采用 page 为单位有几个现实优势：

- 与内核页分配和回收机制自然对齐；
- 不要求为管道准备大块物理连续内存；
- pipe buffer 可用 page 引用、offset 和 length 描述数据；
- 环形缓冲槽位数量适合使用二的幂，便于索引和管理；
- 在不同 page size 架构上可以按同一“页数”组织，而不是依赖固定字节数。

这里的 16 并不是协议正确性的门槛，而是默认资源预算和吞吐需求之间的工程折中。


### 44. 64 KiB 背后的核心理由：在吞吐、内存占用和背压之间折中

内核必须同时兼顾：

```text
缓冲太小：
写入者频繁睡眠和唤醒，读写双方更容易频繁切换，上下文切换和系统调用成本增加。

缓冲太大：
每条 pipe 可能占用更多不可忽视的内核内存；大量进程和管道会放大总资源消耗；慢消费者会积压更多数据，延迟背压出现。
```

16 pages 提供了一个中等规模的突发吸收能力：

- 足以让常见生产者写入一段数据，而不必每几 KiB 就等待；
- 又不会让每条 pipe 默认拥有几 MiB甚至更大的内核缓冲预算；
- 当消费者跟不上时，会较早通过阻塞或 `EAGAIN` 把压力反馈给生产者；
- 应用仍需正确设计边写边读，而不能把 pipe 当无限消息仓库。

因此，64 KiB 的价值不是“保证所有消息都放得下”，而是：

```text
给正常流式通信一定弹性，同时维持有限、可控的内核资源占用和及时背压。
```

Linux 手册公开描述了容量和版本变化，但通常不会把 16 pages 宣称为经过证明的唯一最佳值。把它理解为内核长期演进中形成的默认折中，比理解成某种硬件或协议必需值更准确。


### 45. 为什么系统 RAM 很大，Pipe 也不默认跟着无限增大

假设服务器有 128 GiB RAM，同时创建大量 pipe。如果每条 pipe 默认分配 4 MiB，并有 10,000 条活跃 pipe：

```text
4 MiB × 10,000 = 40,000 MiB，约 39 GiB
```

即使实际页面可能按需分配而不是一创建就全部占满，大容量默认值仍会提高系统最坏情况下的内核内存风险，而且这部分资源由所有进程共同竞争。

更重要的是，大缓冲区并不能解决生产速度长期高于消费速度的问题：

```text
生产 100 MiB/s
消费 10 MiB/s
积压 90 MiB/s
```

缓冲从 64 KiB 调成 4 MiB，只会让阻塞稍晚发生。有限缓冲和阻塞是背压机制，用于迫使生产者最终适应消费者速度，避免“延迟阻塞”演变为无限内存增长。


### 46. 64 KiB 与 `PIPE_BUF = 4 KiB` 完全不是同一个概念

在常见 Linux 系统中经常同时看到：

```text
pipe capacity：常见约 64 KiB
PIPE_BUF：Linux 上通常为 4096 B
```

两者意义不同：

#### Pipe capacity

```text
整条 pipe 当前最多能容纳多少尚未读取的数据。
```

满后，阻塞式 write 会等待；非阻塞 write 可能返回 `EAGAIN`。

#### `PIPE_BUF`

```text
多个写入者并发写同一 pipe 时，单次写入不与其他写入交错的原子性保证上限。
```

`write <= PIPE_BUF` 与“永远不会阻塞”不是同义词。如果 pipe 没有足够空间，阻塞模式仍可能等待。`write > PIPE_BUF` 也不是“不允许写”，只是可能被拆分或与其他写入交错，具体行为还受阻塞模式和可用空间影响。

所以在你的 HTML 场景中：

```text
HTML > 64 KiB
```

主要关联 pipe capacity 和消费者没有及时读取，不是 `PIPE_BUF` 原子写保证。


### 47. 为什么内核不自动把每条 Pipe 无限扩容

自动无限扩容会带来三个问题：

#### 资源失控

慢消费者或完全停止的消费者会让生产者不断把数据堆进内核，最终把通信问题变成系统级内存问题。

#### 公平性下降

少数高速生产者可能占用大量 pipe pages，挤压其他用户和进程可用的内核资源。因此 Linux 提供：

```text
/proc/sys/fs/pipe-max-size
/proc/sys/fs/pipe-user-pages-soft
/proc/sys/fs/pipe-user-pages-hard
```

用于限制单 pipe 和单用户整体 pipe page 使用。

#### 掩盖错误协议

如果双方都采用“先发送全部大对象，再读取”，扩大缓冲只会把死锁触发点从 64 KiB 推迟到 1 MiB、4 MiB 或更高。消息继续增长后仍会失败。

因此 Linux 的策略更接近：

```text
提供有限默认容量；
允许受控调大；
达到上限后实施背压；
要求应用正确消费数据。
```


### 48. 为什么历史上从 1 page 增加到 16 pages

Linux 手册记录：Linux 2.6.11 之前 pipe capacity 通常等于系统 page size；从 Linux 2.6.11 起，容量变为 16 pages。在 4 KiB page 系统上，这就是从约 4 KiB 增加到 64 KiB。

这一变化可以从工程效果上理解为：

```text
1 page 容量很容易在正常数据流中填满，
会造成更频繁的写入阻塞、进程唤醒和调度切换；
扩大到 16 pages 可以提高常见管道工作负载的吞吐和突发容忍度，
同时仍保持有限的每管道内核资源预算。
```

需要谨慎的是：公开 man page 明确记录了“何时从 1 page 变成 16 pages”，但没有给出一条权威表述说“16 是由某个单一公式推导出的绝对最优值”。因此最严谨的结论是：

```text
16 pages 是 Linux 内核实现选择的默认工程折中，
不是物理定律，也不是应用可以依赖的协议常量。
```


### 49. 如何确认当前机器的真实容量

#### 查看 page size 和系统上限

```bash
getconf PAGESIZE
cat /proc/sys/fs/pipe-max-size
cat /proc/sys/fs/pipe-user-pages-soft
cat /proc/sys/fs/pipe-user-pages-hard
```

#### 查询某个实际 pipe

在 Linux 上，可对真正的 pipe file descriptor 使用：

<details>
<summary>展开 Python 代码（9 行）</summary>

```python
import fcntl
import os

read_fd, write_fd = os.pipe()

try:
    size = fcntl.fcntl(write_fd, fcntl.F_GETPIPE_SZ)
    print(size)
finally:
    os.close(read_fd)
    os.close(write_fd)
```

</details>

如果需要调整：

```python
requested = 1024 * 1024
actual = fcntl.fcntl(write_fd, fcntl.F_SETPIPE_SZ, requested)
```

需要注意：

- 这是 Linux 特有接口；
- 未授权进程受 `/proc/sys/fs/pipe-max-size` 和用户 page 限制；
- 内核可能对请求值取整；
- `multiprocessing.Pipe` 的底层对象和全双工模式具有平台实现差异，不能不加验证地把其 file descriptor 当成普通 pipe 调整；
- 即使调整成功，也不能替代“发送期间必须有人读取”的正确协议。


### 50. 对当前问题最有用的理解

不要把当前问题理解为：

```text
Linux 为什么只允许发送 64 KiB？
```

应该理解为：

```text
Linux 为什么只允许约 64 KiB 数据暂时积压在这条通道里，
而不要求接收方立刻读取？
```

答案是：pipe 是有限的内核中转缓冲区，64 KiB 在常见配置下来自 16 个 4 KiB page，是吞吐、内核内存预算和背压之间的默认折中。它允许传输远大于 64 KiB 的总数据，但要求生产者和消费者共同推进。

放回你的父子进程场景：

```text
错误设计：
父进程先 join，不消费；子进程希望先把完整 HTML 发完。

正确设计：
父进程或后台 monitor 在子进程发送期间持续 recv；
通信完成后再 join 和清理。
```

所以真正需要修复的是控制流和资源所有权，而不是把 64 KiB 当成异常限制去绕过。


### 51. 本部分参考资料

1. Python documentation, `multiprocessing — Process-based parallelism`\
   https://docs.python.org/3/library/multiprocessing.html
2. Linux man-pages, `pipe(7) — overview of pipes and FIFOs`\
   https://www.man7.org/linux/man-pages/man7/pipe.7.html
3. Linux man-pages, `F_GETPIPE_SZ / F_SETPIPE_SZ`\
   https://www.man7.org/linux/man-pages/man2/F_SETPIPE_SZ.2const.html
4. Linux man-pages, `pipe(2) — create pipe`\
   https://www.man7.org/linux/man-pages/man2/pipe.2.html




---

</details>
