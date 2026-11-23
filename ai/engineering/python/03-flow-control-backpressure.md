# 03 流控、背压与端到端稳定性

> 原稿最早可见 Git 记录：2026-10-14；原稿最近 Git 修改：2026-10-15。公开版整理：2026-11-17。以上 Git 日期用于追溯，不等同于实际首次写作日期。
> 本文为通用知识整理，历史产品能力及版本细节需结合当前文档复核。

章节编号沿用原稿，便于在系列中回溯；长代码与推导可折叠阅读。

## 深入：流控与背压

**与本问题直接相关的结论：**Pipe 满时的阻塞是背压信号，不是传输失败。必须有活跃接收者持续消费；扩大缓冲只能吸收更大的短期突发，不能解决接收方永久缺席。

<details>
<summary>流控与背压详解：Pipe、TCP、HTTP/2、gRPC 和 Kafka（第 52–71 节）</summary>

### 52. 背压：阻塞其实是系统的保护机制

假设：

```text
生产者速度：100 MiB/s
消费者速度：1 MiB/s
```

如果系统无限缓存，每秒积压约 99 MiB，一分钟接近 6 GiB。有限缓冲区填满后阻塞生产者，迫使生产速度受到消费能力约束，这就是背压。

```text
消费者变慢
    ↓
缓冲区逐渐填满
    ↓
生产者 write/send 阻塞或收到 EAGAIN
    ↓
生产速率被压回可承受范围
```

因此，“缓冲区满后发送阻塞”本身不是系统缺陷。应用需要做到以下之一：

- 接收端及时消费；
- 使用非阻塞 I/O 并处理 `EAGAIN`；
- 使用 `select`、`poll` 或 `epoll` 等事件机制；
- 对待传输任务建立有界队列；
- 将大数据放在文件或共享内存中，只传元数据；
- 设计分块协议和流量控制。


### 53. 为什么这两个概念容易混淆

`Flow Control` 和 `Back Pressure` 经常同时出现，因为二者都在解决同一个基本矛盾：

```text
上游产生或发送数据的速度
>
下游接收、处理或持久化数据的速度
```

如果系统不采取措施，速度差只能暂存在某个地方：

```text
生产者
   ↓ 100 MB/s
[内存队列 / Pipe / Socket Buffer / Broker]
   ↓ 10 MB/s
消费者
```

每秒净积压：

```text
100 MB - 10 MB = 90 MB
```

只要运行足够久，任何有限缓冲区都会被填满。因此，一个稳定系统不能只回答“数据暂时放在哪里”，还必须回答：

```text
当下游处理不过来时，如何让上游减速、暂停、拒绝、重试或丢弃？
```

这就是 Back Pressure 要表达的核心。


### 54. Back Pressure 最直观的定义

Back Pressure 可以翻译为“反压”或“背压”。它不是一种特定协议，也不是某个固定 API，而是一种系统控制现象：

> 当下游无法继续按当前速度接收或处理数据时，压力沿着数据流的反方向传回上游，迫使上游减速、暂停、等待、拒绝新任务或采取降级策略。

数据流方向：

```text
上游 Producer ───── 数据 ─────> 下游 Consumer
```

压力反馈方向：

```text
上游 Producer <── 减速信号 ─── 下游 Consumer
```

所以名称中的 `Back` 指的是：压力传播方向与正常数据流方向相反。

#### 水管类比

```text
水泵        管道        水箱出口
上游        缓冲区       下游
```

如果水泵每秒送 100 升，而出口每秒只能排 10 升：

```text
水位持续上升
    ↓
水箱装满
    ↓
系统必须让水泵减速或停机
```

“水箱逐渐装满”只是积压；“水箱满后让水泵减速”才是背压。

因此要区分：

```text
Backlog：已经积压了多少数据。
Back Pressure：积压达到限制后，系统如何把压力反馈给生产者。
```


### 55. Back Pressure 不是错误，而是稳定系统的必要反馈

很多开发者第一次看到：

```text
send() 阻塞
producer.send() 变慢
stream write 暂停
```

会认为系统坏了。但如果消费者确实处理不过来，上游仍然无限发送，替代结果通常只有：

- 内存持续上涨直至 OOM；
- 队列无限增长，延迟越来越大；
- 磁盘耗尽；
- 连接和文件描述符耗尽；
- 大量超时、重试和重复数据形成重试风暴；
- 最终整个系统崩溃，而不只是生产者暂时变慢。

Back Pressure 的作用不是让系统永远不慢，而是让“容量不足”以受控方式表现出来：

```text
无背压：
过载 → 无限积压 → OOM / 崩溃 / 数据丢失

有背压：
过载 → 阻塞 / 减速 / 拒绝 / 限流 → 系统保持有界
```

它传达的是重要事实：

```text
系统整体吞吐量最终受最慢环节限制。
```

缓冲区只能吸收短期突发，不能永久消除上下游的长期速度差。


### 56. Back Pressure 可以怎样向上游表达

反压不一定表现为线程阻塞。常见形式包括：

#### 1. 阻塞发送

```python
send(data)  # Wait until downstream capacity becomes available.
```

Linux 满 Pipe 上的阻塞式 `write()`、gRPC 某些语言中的 stream write 阻塞，都属于这一类。

#### 2. 返回“暂时不可写”

非阻塞 I/O 中：

```text
write/send → EAGAIN 或 EWOULDBLOCK
```

应用收到后不能无限紧循环重试，而应等待 `poll/epoll` 通知可写。

#### 3. 使用窗口或配额

接收方告诉发送方：

```text
你目前最多还能发送 N 字节或 N 条消息。
```

TCP receive window、HTTP/2 stream/connection window 都属于这种 credit-based flow control。

#### 4. 暂停拉取

消费者主动减少或停止拉取新任务：

```text
本地工作队列已满
    ↓
暂停从 Broker 拉取
    ↓
下游恢复后继续拉取
```

#### 5. 有界队列阻塞或拒绝

```text
queue 有容量 → 接收任务
queue 已满 → 阻塞、超时、返回失败或拒绝
```

#### 6. HTTP 限流或过载响应

服务过载时返回：

```text
429 Too Many Requests
503 Service Unavailable
Retry-After
```

这是一种跨请求级别的显式反压，但客户端必须尊重退避，才会真正形成闭环。

#### 7. 降低生产速率或采样

对于监控、日志或遥测数据，可以：

- 降低采样频率；
- 聚合数据；
- 丢弃低优先级事件；
- 合并重复状态；
- 优先保证关键流量。

这属于业务级背压策略。


### 57. Flow Control 与 Back Pressure 的关系和区别

可以这样理解：

```text
Back Pressure：系统层面的目标和反馈现象。
Flow Control：实现该目标的一类具体机制，通常控制发送方当前允许发送多少数据。
```

二者并非所有语境下都有绝对统一的边界，但在网络协议中常见区分是：

#### Flow Control

重点保护接收方：

```text
接收方还有多少 buffer？
接收应用读取有多快？
发送方当前最多还能发多少？
```

#### Back Pressure

概念更宽，可以跨越多层系统：

<details>
<summary>展开时序/结构示意（11 行）</summary>

```text
数据库变慢
    ↓
业务 worker 堆积
    ↓
服务暂停读取 RPC stream
    ↓
gRPC/HTTP/2 window 不再快速恢复
    ↓
客户端 write 变慢或阻塞
    ↓
客户端降低任务生成速度
```

</details>

这里压力从数据库一路反向传播到最上游生产者。TCP 或 HTTP/2 Flow Control 只是这条反压链中的一环。

#### 一个重要判断

有 Flow Control 不代表应用一定正确处理了 Back Pressure。

例如框架暂时在内存里缓存消息，应用的 `send()` 每次都立即返回：

```text
应用以为下游很快
框架队列却持续膨胀
```

如果框架最终没有阻塞、拒绝或向应用暴露 readiness，那么压力没有完整传回真正生产者，只是在中间层被延迟。


### 58. Flow Control 与 Congestion Control 也不是一回事

这两个概念在 TCP 中尤其容易混淆。

#### Flow Control 保护接收方

问题是：

```text
接收端应用和接收缓冲区能不能处理这么快？
```

关键变量可概念化为 TCP receive window，即 `rwnd`。

#### Congestion Control 保护网络

问题是：

```text
网络路径中的链路、交换机、路由器是否拥塞？
```

关键变量可概念化为 congestion window，即 `cwnd`。

发送方实际可发送的在途数据通常同时受两者约束：

```text
allowed_in_flight ≈ min(rwnd, cwnd)
```

含义是：

```text
接收方只能收 64 KiB，即使网络不拥塞，也不能超发；
网络只能承受更少数据，即使接收方 buffer 很大，也要因拥塞减速。
```

TCP 标准规定了接收窗口等传输行为；TCP 的流量控制与网络拥塞控制解决的是不同瓶颈。


### 59. Linux Pipe 中的 Back Pressure

你的父子进程问题就是最直接的背压示例：

```text
子进程：Producer，生成并发送 HTML
父进程：Consumer，接收 HTML
Pipe：有限内核缓冲区
```

正常路径：

```text
子进程 write/send
    ↓
Pipe 暂存数据
    ↓
父进程 read/recv
```

父进程读取速度跟不上时：

```text
Pipe occupancy 上升
    ↓
Pipe 满
    ↓
子进程 write/send 阻塞
```

这个阻塞就是 Pipe 对子进程施加的反压：

```text
父进程暂时没有消费能力，请停止继续生产或发送。
```

问题不在于背压出现，而在于父进程当时阻塞在 `join()`，所以它永远不会执行能够解除背压的 `recv()`：

```text
正常背压：
消费者变慢 → 生产者暂时阻塞 → 消费者读取 → 生产者恢复

你的死锁：
消费者完全不读取并等待生产者退出
→ 生产者因缓冲区满而等待消费者读取
→ 双方永久等待
```

因此先 `recv()` 后 `join()` 的本质，不只是“交换顺序”，而是恢复正确的背压闭环：

```text
Pipe 满 → 子进程暂停
父进程 recv → Pipe 释放空间
子进程继续发送
```


### 60. TCP 如何实现 Flow Control 和 Back Pressure

TCP 是字节流协议。接收端操作系统为连接维护 receive buffer，应用通过 `recv()` 从该 buffer 取数据。

```text
发送应用
   ↓
发送端 TCP
   ↓ network
接收端 TCP receive buffer
   ↓
接收应用 recv()
```

接收端通过 TCP header 中的窗口信息向发送端通告还能接收多少数据。发送端不能让受流量控制约束的未确认数据超过接收方通告能力。现代 TCP 还可以通过 Window Scale 扩展有效窗口；RFC 9293 是当前 TCP 核心规范。\
来源：https://www.rfc-editor.org/rfc/rfc9293.html

#### 接收应用很快

```text
应用持续 recv()
    ↓
receive buffer 被及时排空
    ↓
接收窗口保持较大
    ↓
发送方可以持续发送
```

#### 接收应用很慢

```text
应用不及时 recv()
    ↓
receive buffer 越来越满
    ↓
接收方通告更小窗口
    ↓
发送方减少允许的在途数据
```

#### 接收 buffer 完全满

接收方可以通告 zero window：

```text
rwnd = 0
```

发送方暂停正常数据发送，直到收到窗口更新。最终这种限制会向上传播到发送端应用：发送端 socket buffer 也会被填满，阻塞式 `send()` 可能阻塞，非阻塞 `send()` 可能返回暂时不可写。

所以完整反馈链为：

<details>
<summary>展开时序/结构示意（11 行）</summary>

```text
接收应用变慢
    ↓
接收 TCP buffer 填满
    ↓
TCP receive window 缩小或归零
    ↓
发送端 TCP 暂停/减速
    ↓
发送端 socket buffer 填满
    ↓
发送应用的 send() 阻塞或 EAGAIN
```

</details>

这就是压力跨越网络从接收应用反向传回发送应用。

#### TCP Flow Control 的局限

TCP 只知道字节和 buffer，不理解业务语义：

```text
它不知道一段 bytes 是 HTML、订单还是模型结果；
不知道数据库是否已经提交；
不知道业务线程是否真正处理完成。
```

如果服务端持续把网络数据读入自己的无界应用队列，TCP 会认为接收端有能力，窗口保持开放，但应用内存仍可能不断增长。要形成端到端背压，应用层也必须使用有界队列、暂停读取或拒绝请求。


### 61. HTTP/2 为什么还要单独做 Flow Control

HTTP/2 通常运行在 TCP 上。既然 TCP 已有 Flow Control，HTTP/2 为什么还需要自己的机制？因为一条 HTTP/2 TCP connection 可以复用多个逻辑 stream：

```text
一个 TCP connection
├── Stream 1：下载大文件
├── Stream 3：API 请求
├── Stream 5：流式响应
└── Stream 7：另一个 RPC
```

TCP 只能看见整个连接的字节流，不知道哪些 bytes 属于哪个 stream。HTTP/2 因而提供两层 flow-control window：

```text
Connection-level window：限制整个 HTTP/2 连接的 DATA 总量。
Stream-level window：限制某个具体 stream 的 DATA 量。
```

HTTP/2 RFC 9113 专门定义了 Flow Control 原则、使用方式和性能考虑，并使用 `WINDOW_UPDATE` 增加发送额度。HTTP/2 的流量控制作用于 DATA frame，不是所有 frame。\
来源：https://www.rfc-editor.org/rfc/rfc9113.html

#### Credit 模型

把 window 想成接收方发给发送方的“可消费额度”：

```text
Stream 1 初始额度：65,535 bytes
发送方发送 20,000 bytes
剩余额度：45,535 bytes
接收方处理后发送 WINDOW_UPDATE +20,000
额度恢复：65,535 bytes
```

如果额度耗尽：

```text
该 stream 暂停发送 DATA
```

接收方处理并发送 `WINDOW_UPDATE` 后，发送方才能继续。

#### 为什么需要 stream-level window

假设大文件 Stream 1 的接收应用很慢，而 Stream 3 是一个小 API 响应。如果只有 TCP connection 级别控制，大流量 stream 更容易占用整个连接的接收能力。Stream-level window 让协议能够单独限制某个 stream，而不只是把整条连接视为一个不可区分的字节桶。

#### HTTP/2 Flow Control 仍可能死锁

如果双方都：

```text
只写大量 DATA
不读取对方的数据
等待对方先完成
```

双方的 flow-control window 都可能耗尽并停止前进。这与父子进程“双方先发完再收”是同类协议错误。Flow Control 能提供安全减速，但不能自动修复错误的应用读写顺序。


### 62. gRPC 中 Flow Control 和 Back Pressure 怎样体现

gRPC 常使用 HTTP/2 作为传输，因此继承 HTTP/2 stream 和 connection flow control。gRPC 官方文档将 Flow Control 定义为防止快速 sender 压垮 receiver 的机制，主要与 streaming RPC 相关；框架通常自动管理，但某些语言允许手动控制。官方文档也明确提醒，如果客户端和服务端都使用同步读写或手动 flow control，并且双方只做大量写入而不读取，可能死锁。\
来源：https://grpc.io/docs/guides/flow-control/

#### Unary RPC 与 Streaming RPC

Unary RPC：

```text
一个 request → 一个 response
```

应用层通常较少直接看到连续流量控制过程。

Streaming RPC：

```text
client stream
server stream
bidirectional stream
```

发送方可以连续产生许多消息，更容易出现“生产快于消费”的长期速度差，所以 Flow Control 尤为重要。

#### `write()` 返回不代表对方业务已经处理

典型分层：

<details>
<summary>展开时序/结构示意（15 行）</summary>

```text
发送应用调用 write(message)
    ↓
gRPC framework 接收消息
    ↓
序列化和内部 buffering
    ↓
HTTP/2 flow-control window
    ↓
TCP socket buffer
    ↓
网络
    ↓
接收端 gRPC framework
    ↓
接收应用 read/process
```

</details>

所以某次 `write()` 返回可能只表示框架接受了消息，不表示：

- 消息已经离开发送主机；
- 对端已经收到；
- 对端应用已经读取；
- 对端业务已经处理完成。

当 flow-control 额度耗尽后，框架可能等待、延迟 write completion，或通过 readiness API 告知暂时不可继续发送，具体表现取决于语言实现。gRPC-Go 的官方示例说明，当 stream flow-control limit 达到时，阻塞式 send 会暂停；接收方读取足够数据后，发送才解除阻塞。\
来源：https://pkg.go.dev/google.golang.org/grpc/examples/features/flow_control

#### gRPC 中真正的端到端背压

假设服务端收到消息后写数据库：

<details>
<summary>展开时序/结构示意（11 行）</summary>

```text
数据库变慢
    ↓
服务端处理消息速度下降
    ↓
服务端减少 read/request 新消息的速度
    ↓
gRPC/HTTP/2 接收额度恢复变慢
    ↓
客户端 send/write 变慢或进入 not-ready
    ↓
客户端停止继续从本地任务源生成无限消息
```

</details>

如果服务端虽然数据库很慢，却仍然快速从 gRPC stream 读取消息并放进无界内存队列，那么 HTTP/2 会持续认为它有接收能力，协议层反压被应用层截断，最终仍可能 OOM。


### 63. Kafka 如何体现 Back Pressure

Kafka 与 Pipe/TCP 有一个重要差异：Kafka 本身是持久化日志，允许生产者和消费者在时间上解耦。因此 Kafka 可以把一部分速度差变成磁盘上的 consumer lag，而不是立即阻塞每个生产者。

```text
Producer → Kafka Broker 持久化日志 → Consumer
```

#### Producer 侧背压

Kafka producer 通常先把 records 放进客户端内存 buffer，再由后台发送线程批量发送到 broker：

```text
应用 producer.send()
    ↓
Record Accumulator / buffer.memory
    ↓
后台 Sender
    ↓
Broker
```

如果应用产生数据的速度持续高于 broker 接收速度：

```text
producer buffer 越来越满
    ↓
buffer.memory 耗尽
    ↓
发送调用最多等待 max.block.ms
    ↓
仍无空间则失败并抛出异常
```

Apache Kafka 官方 producer 配置说明，`buffer.memory` 是 producer 用于缓冲待发送 records 的总字节预算；如果 records 产生速度高于发送到 server 的速度，producer 会阻塞最多 `max.block.ms`，之后失败。\
来源：https://kafka.apache.org/41/configuration/producer-configs/

这就是明确的 producer-side back pressure：broker 或网络变慢，压力经过 producer buffer 反向传回应用调用者。

#### Consumer 侧并不总是直接反压 Producer

如果 consumer 处理变慢：

```text
Broker 中未消费 offset 增加
    ↓
Consumer Lag 上升
```

Kafka 可以依靠 broker 磁盘暂存 backlog，因此短时间内 producer 未必立即变慢。这里需要特别准确地说：

```text
Consumer Lag 是积压指标，不等于压力已经传回 Producer。
```

如果 producer 继续稳定写入而 consumer 永远更慢，最终问题会表现为：

- lag 持续增加；
- end-to-end latency 增加；
- retention 到期前来不及处理，产生逻辑数据丢失；
- broker 磁盘、partition 或集群吞吐成为瓶颈；
- broker 限流或 producer buffer 最终让压力传回生产应用。

因此 Kafka 把即时背压转换成“可持久化、可观察、可扩容的积压”，但并没有消除容量守恒。

#### Kafka 消费应用仍需本地背压

如果 consumer poll 很快，却把 records 放进无界线程池队列：

```text
Kafka lag 看起来可能不高
应用内存队列却不断增长
最终 consumer 进程 OOM
```

正确方式需要有界工作队列，并在本地容量不足时暂停拉取、降低并发或控制 poll 与处理节奏。否则只是把积压从 broker 磁盘搬到了更脆弱的进程内存。


### 64. 四种机制放在一起比较

#### Linux Pipe

```text
容量位置：内核 Pipe buffer
反馈方式：满时阻塞 write/send，或非阻塞返回 EAGAIN
保护对象：本机读取进程与内核内存
你的问题：父进程 join 导致无人读取，正常背压升级为死锁
```

#### TCP

```text
容量位置：收发 socket buffer
反馈方式：receive window 缩小或归零，最终让发送应用阻塞/EAGAIN
保护对象：远端接收 buffer 和应用读取能力
粒度：一条 TCP connection 的字节流
```

#### HTTP/2 / gRPC

```text
容量位置：connection/stream window 及框架 buffer
反馈方式：额度耗尽、WINDOW_UPDATE、write 等待或 readiness
保护对象：单个逻辑 stream 和整条 multiplexed connection
粒度：比 TCP 更理解逻辑 stream，但仍不理解最终业务是否提交
```

#### Kafka

```text
容量位置：producer memory buffer + broker 持久化日志 + consumer 本地队列
反馈方式：producer block/timeout、broker throttling、consumer pause、lag 告警
保护对象：producer、broker 和 consumer 各层资源
特点：速度差可以暂时持久化为 lag，不一定立即反压到生产者
```

共同规律：

```text
所有系统都只能做三件事的组合：
1. 暂存；
2. 减速或拒绝；
3. 丢弃或降级。

不存在无限速度差下仍能无限无损运行的第四种方案。
```


### 65. Buffering 与 Back Pressure 的区别

缓冲是“先吸收”，反压是“吸收不下后怎么办”。

```text
突然突发 100 MB
消费者稍后可以追上
→ 足够 buffer 可以平滑突发

生产者永久比消费者快 90 MB/s
→ 任何有限 buffer 最终都会满
→ 必须反压、扩容、降级或丢弃
```

扩大 buffer 的作用：

- 提高短期突发容忍度；
- 降低频繁阻塞和唤醒；
- 可能提高吞吐。

扩大 buffer 的代价：

- 增加内存或磁盘占用；
- 增加排队延迟；
- 推迟故障暴露；
- 可能让过载时需要处理更多陈旧任务。

因此 buffer sizing 和 back-pressure design 是两个相关但不同的决策。


### 66. 背压如何与吞吐、延迟和稳定性取舍

假设消费者最多处理 100 tasks/s，而入口短期达到 150 tasks/s。

#### 无限队列

```text
入口不受影响
队列每秒增加 50 条
延迟持续增长
最终内存耗尽
```

#### 小型有界队列 + 阻塞

```text
队列很快填满
生产者被迫降到接近 100 tasks/s
内存稳定
入口延迟或调用阻塞增加
```

#### 有界队列 + 快速拒绝

```text
超过容量立即 429/503
系统内部延迟和内存稳定
客户端必须退避或接受失败
```

#### 持久化队列

```text
短期突发写入磁盘
消费者稍后追赶
积压期间业务端到端延迟增加
长期超载仍会耗尽 retention 或磁盘
```

没有一种策略在所有维度都最好。选择取决于：

- 是否允许阻塞调用者；
- 是否允许丢数据；
- 是否允许高延迟；
- 数据是否可以重放；
- 是否有磁盘持久化；
- 上游能否重试；
- 下游能否水平扩展；
- 是否存在优先级流量。


### 67. 为什么错误重试会破坏 Back Pressure

下游过载时返回失败，本来是在要求上游减速。如果上游立即、无限、无抖动重试：

<details>
<summary>展开时序/结构示意（9 行）</summary>

```text
下游过载
    ↓
大量请求失败
    ↓
所有客户端立即重试
    ↓
流量比原来更大
    ↓
下游更加过载
```

</details>

这叫 retry storm。正确的配套措施通常包括：

- exponential backoff；
- random jitter；
- retry budget；
- 最大重试次数；
- deadline；
- circuit breaker；
- `Retry-After`；
- 幂等性设计。

所以“返回错误”只有在上游尊重反馈时，才形成真正的反压闭环。


### 68. 如何判断系统有没有完整的端到端 Back Pressure

沿数据链逐层追问：

```text
入口 → API → 内存队列 → Worker → 数据库/外部服务
```

对每一层问：

1. 缓冲区是否有上限？
2. 满了之后是阻塞、拒绝、丢弃还是继续扩容？
3. 压力是否会传回真正的数据生产者？
4. 上游收到信号后是否真的会减速？
5. 任务是否有 deadline 和取消机制？
6. 重试是否带退避和上限？
7. 是否区分高优先级与可丢弃数据？
8. 能否监控 queue depth、lag、window、blocked time 和 rejection rate？

典型的“假背压”是：

```text
网络层暂停了一下，
应用却继续把任务放进另一个无界内存队列。
```

这只是把压力移动了位置，没有真正约束系统总在途数据。


### 69. 回到你的父子进程方案：Back Pressure 视角下的设计判断

你的数据链可以写成：

<details>
<summary>展开时序/结构示意（9 行）</summary>

```text
子进程 generate_html()
    ↓
child_conn.send(html)
    ↓
有限 IPC buffer
    ↓
parent_conn.recv()
    ↓
父进程处理/保存/返回
```

</details>

正确闭环必须保证：

```text
1. 子进程发送太快时，可以被 Pipe 暂时阻塞；
2. 父进程或后台 monitor 会持续读取，最终解除阻塞；
3. HTTP 返回 202 后，读取责任不能消失；
4. 如果父进程不再愿意接收，必须关闭通道并让子进程明确失败或取消；
5. 所有终态都要更新任务状态并回收子进程。
```

所以前面提出的关键问题现在可以用 Back Pressure 的语言重新表达：

> 202 返回后，如果没有 monitor 继续 `recv()`，背压就无法被解除。Pipe 的有限缓冲会正确地让子进程停止发送，但系统中没有任何消费者恢复读取，于是受控减速变成永久阻塞。

#### 是否应该调大 Pipe

调大 Pipe 只是在反馈链中增加暂存空间：

```text
64 KiB → 1 MiB
```

它可以吸收更大的短期突发，但不会改变：

```text
最终必须有人 recv；
长期没有消费者时，任何有限容量都会满。
```

因此，当前设计决策的优先级应是：

```text
第一：明确 202 后的消费者和资源所有权；
第二：正确处理 recv/join、EOF、OOM 和终态；
第三：测量后再决定是否为了性能调大 buffer 或改为文件/持久化结果。
```


### 70. 一个最简记忆模型

记住三个词即可：

```text
Buffer：先存一点。
Flow Control：规定还能发多少。
Back Pressure：下游处理不过来时，让上游真正慢下来。
```

再记住一个守恒关系：

```text
长期稳定吞吐量
≤
系统中最慢关键环节的处理能力
```

缓冲、批处理和并发可以改善短期效率，但不能推翻这个关系。


### 71. 本部分参考资料

1. RFC 9293, Transmission Control Protocol (TCP)\
   https://www.rfc-editor.org/rfc/rfc9293.html
2. RFC 9113, HTTP/2，特别是 Section 5.2 Flow Control\
   https://www.rfc-editor.org/rfc/rfc9113.html
3. gRPC official guide, Flow Control\
   https://grpc.io/docs/guides/flow-control/
4. gRPC-Go flow-control example\
   https://pkg.go.dev/google.golang.org/grpc/examples/features/flow_control
5. Apache Kafka Producer Configs，`buffer.memory` 与 `max.block.ms`\
   https://kafka.apache.org/41/configuration/producer-configs/
6. Linux man-pages, `pipe(7)`\
   https://www.man7.org/linux/man-pages/man7/pipe.7.html




---

 </details>
