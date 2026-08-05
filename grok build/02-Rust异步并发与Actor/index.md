# 02 Rust 异步、并发与 Actor

这一章是给没有写过异步 Rust 的读者准备的。后面大量源码会出现 `tokio::spawn`、`mpsc`、`LocalSet`、`JoinSet`、`watch` 和 `CancellationToken`。如果不先知道它们在解决什么问题，很容易把“多任务协作”看成一堆随机的 channel。

## async 不等于开了一个线程

`async fn` 返回的是一个 future，遇到 `.await` 时可以把控制权交回 Tokio runtime，让别的任务运行。它适合等待网络、文件或 channel；它不会自动把 CPU 密集计算搬到另一个线程。

可以先用这句不严谨但有帮助的话理解：

> 线程是房间，async task 是房间里的协作流程；`.await` 是流程暂时让出房间。

Grok Build 的模型请求、socket、TUI 输入、session command 都有等待 I/O 的部分，所以 Tokio 能把大量等待组织在同一个 runtime 里。需要独立线程时，代码会显式使用 OS thread 或 `spawn_blocking`。

## channel 让模块不直接互相改状态

常见 Tokio channel：

| 类型 | 用途 | 你会在项目里看到的场景 |
| --- | --- | --- |
| `mpsc` | 多个发送者给一个接收者发命令 | `SessionHandle` 控制 `SessionActor` |
| `oneshot` | 一次请求对应一次回复 | submit 后拿 response/metrics |
| `watch` | 只关心最新值 | disconnect reason、配置状态 |
| `broadcast` | 多个订阅者收到事件 | 通知或状态分发 |

`SessionHandle` 不直接持有 `SessionActor` 的所有字段，而是持有 sender。调用方只能发一个定义好的 `SessionCommand`，这就是控制面和执行面分离的第一步。

## actor 在这里不是框架魔法

Grok Build 的 actor 通常由四部分构成：

1. 一个拥有状态的结构体。
2. 一个接收命令的 channel。
3. 一个循环，用 `select!` 等待命令、完成的子任务和外部事件。
4. 一个给外部使用的 handle，隐藏 sender 和回复方式。

**伪代码**如下：

```text
state = 创建 actor 状态
loop:
    等待 Command 或 ChildTaskFinished 或 Shutdown
    Command::Prompt => 修改 state，启动/等待一次 turn
    ChildTaskFinished => 把结果写回 state
    Shutdown => 保存状态，退出
```

`SamplerActor` 和 `SessionActor` 都有这个形状，但它们拥有的状态不同。不要因为都叫 actor，就认为它们能互换。

## `LocalSet` 和 `!Send` 为什么出现

[`spawn.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/session/acp_session_impl/spawn.rs) 会为 session 建立专用 OS thread、Tokio runtime 和 `LocalSet`，让 `!Send` 的 `SessionActor` 固定在该线程上。简单说，`Send` 表示值能安全地跨线程移动；不满足它的状态必须留在同一个线程。

这不是“性能优化”四个字就能解释完的决定。它让某些本地状态不用到处加锁，也要求通过 `SessionHandle` 跨线程通信。读这类代码时，先标出状态在哪个线程拥有，再看 sender/receiver 怎样跨过去。

## 取消是合作式的

`CancellationToken` 发出取消信号，不会像操作系统杀进程那样立刻中断所有代码。每个任务需要在合适的 `select!` 或 `.cancelled()` 分支里观察它，然后关闭请求、清理 terminal、发送 completion 或释放锁。

所以一个“取消 turn”的完整问题是：

- sampler 请求是否停止读模型 stream？
- 当前工具进程是否停止或等待退出？
- session 是否写入 `Cancelled` completion？
- TUI 是否继续接收残余输出？

后面第 12 章会沿这四个问题回到真实调用链。

## 一个 actor 到底解决了什么

把状态集中到 actor 里，不是为了让代码看起来“更异步”，而是为了减少多个任务同时修改同一份状态的机会。外部拿到的是 `Handle` 或 `Sender`，只能提交命令；真正的 `SessionActor` 或 `SamplerActor` 在自己的循环里决定顺序。

这个模型让几个问题更容易回答：

| 问题 | 看哪里 |
| --- | --- |
| 谁拥有可变状态 | actor struct 的字段和初始化函数 |
| 谁能改变状态 | command enum 的 match 分支 |
| 谁等待外部 I/O | `select!`、request task、stream reader |
| 谁把结果送回去 | oneshot responder、event sender、watch state |

但 actor 不是魔法。一个 actor 可以把工作转交给子 task；子 task 如果绕过 actor 直接写共享状态，串行假设就失效。读源码时要沿 `spawn` 继续追，不要看到一个 `send()` 就认为所有逻辑都在 receiver 线程上完成。

## `select!` 里的优先级和取消

`tokio::select!` 表示多个 future 同时等待，但“谁先完成”还会受 branch 顺序、biased 设置、取消安全和外层 token 影响。session loop 可能同时等 prompt、tool event、watcher、timer 和 shutdown；TUI loop 还要等 keyboard、ACP、draw request 和 writer event。

遇到取消时，我会逐层问：

1. 哪一个 token 被 cancel？是当前 turn，还是整个 session？
2. 哪些 future 只是停止等待，哪些外部资源需要显式 kill、close 或 flush？
3. 已经产生的 delta、tool result 和 usage 是否写入历史？
4. 调用方收到的是 `Cancelled`、`Failed`、`Incomplete` 还是正常 completion？

这四个答案可能不一致。网络 request 停了，不代表工具进程停了；UI 不再重绘，也不代表 session 没有继续收尾。

## `!Send` 为什么需要 `LocalSet`

`Send` 可以先理解为“值能安全地移动到另一个线程”。某个 actor 如果持有不能跨线程移动的资源，就不能随便用 `tokio::spawn` 把它丢到线程池。`LocalSet` 让一组非 `Send` future 固定在同一条线程上运行；handle 仍然可以从别的线程发送命令。

这带来一个很实际的阅读提醒：不要把“有 Tokio runtime”理解成“所有任务都在同一个地方”。源码可能有 pager runtime、leader runtime、session 专用 runtime 和请求 task。先画线程/Runtime，再画 channel，才能判断一个锁或 token 的作用范围。

## 小练习：把命令改写成时序

从 `Prompt` 之外再选 `SetSessionModel`、`CompactSession` 或 `StopTask`。按下面的格式记录：

```text
调用方 -> Handle::send(command)
      -> actor loop 收到 command
      -> 状态变化 / 启动子 task
      -> responder 返回一次结果
      -> event channel 广播可观察事件
```

如果某一步找不到，不要补一个想象中的函数名；写下“当前快照尚未确认”。这会比把异步代码翻译成一条过于漂亮的直线更可靠。

## 小实验

不运行 Grok Build，也能练习读 actor：

```bash
rg -n "mpsc::|CancellationToken|tokio::select!|JoinSet|LocalSet" \
  crates/codegen/xai-grok-shell/src/session \
  crates/codegen/xai-grok-sampler/src/actor
```

任选一个 receiver，写下它能收到的命令；再找出每个命令在哪里发出。能画出“谁发—谁收—谁回”的三列表，就已经跨过后面源码阅读的第一道门槛。
