# 00 阅读前先建立概念

这份笔记不是 Rust 语法教程，也不是 LLM 产品使用手册。它假设你愿意读一点代码，但不假设你已经知道 Agent 项目里的术语。为了不让后面的 `SessionActor`、`SamplerHandle` 和 ACP 把人挡在门外，我先把词汇翻译成人话。

本单元里的源码名词以 [Grok Build 的 `ed6d543`](https://github.com/xai-org/grok-build/commit/ed6d543643628663873c5de28298e022ed634238) 快照为准；这里先讲阅读地图，后面的单元会把每个词落到具体文件。

## 先把 Agent 想成一个会做事的程序

普通聊天程序大致是：用户发消息 → 调模型 → 显示回答。

coding agent 多了一层循环：模型不仅输出文字，还可以请求工具；程序执行工具，把结果放回对话，再让模型继续判断。工具可能是读文件、改文件、执行 shell、搜索网页、调用 LSP 或启动另一个子 agent。

下面是一个**伪代码**，不是 Grok Build 的源码：

```text
把用户消息加入 conversation
重复：
    把 conversation 发给模型
    如果模型只返回文字：显示文字，结束本轮
    如果模型返回工具调用：
        检查参数和权限
        执行工具
        把工具结果加入 conversation
        继续请求模型
```

Grok Build 的复杂之处在于，这几行伪代码被拆到很多 crate 和 actor 里，而且还要支持取消、并发工具、会话恢复、多个客户端、终端回压、上下文压缩和安全策略。

## 这些词分别指什么

| 词 | 白话解释 | 在源码里通常寻找什么 |
| --- | --- | --- |
| model | 接收上下文并返回文本/工具调用的模型 | request、response、stream、usage |
| prompt | 用户或系统给模型的输入 | `ConversationRequest`、system prompt |
| tool | Agent 可以请求程序执行的动作 | schema、tool call、tool result |
| turn | 从一次 prompt 到这一轮完成的过程 | `turn.rs`、completion kind |
| session | 可保存、恢复和继续的会话 | history、persistence、session id |
| workspace | Agent 当前操作的代码目录和主机能力 | cwd、filesystem、VCS、permissions |
| TUI | Terminal User Interface，终端里的交互界面 | `ratatui`、render、event loop |
| ACP | Agent Client Protocol，让外部客户端接入 Agent | `AgentSideConnection`、JSON-RPC |
| leader | 负责复用 Agent runtime 的常驻进程 | Unix socket、register、reconnect |
| actor | 拥有自己的状态、通过消息接收命令的任务 | `mpsc`、command、event、handle |
| compaction | 历史太长时，用摘要替代一部分历史 | token budget、summary、checkpoint |
| memory | 跨 session 保存和检索的知识 | `~/.grok/memory`、FTS、embedding |

## 读 Rust 源码时最低限度要认识的东西

| Rust 概念 | 可以先怎么理解 | Grok Build 里的例子 |
| --- | --- | --- |
| crate | 一个可编译的库或二进制包 | `xai-grok-sampler`、`xai-grok-pager-bin` |
| module | crate 内的源码分组 | `session::acp_session_impl` |
| trait | 对一类类型要求的行为接口 | `Tool`、workspace operation |
| `struct` / `enum` | 带字段的数据；有限的状态选择 | `SessionHandle`、`Decision` |
| `Arc` | 多个任务共享同一份引用计数状态 | 工具资源和 session context |
| `async` / `await` | 不阻塞当前线程地等待 I/O | 模型请求、socket、文件操作 |
| channel | 一个任务给另一个任务发消息的队列 | Tokio `mpsc`、oneshot、watch |
| `select!` | 同时等待多个事件，谁先到谁处理 | session loop、TUI loop |
| `CancellationToken` | 给异步任务发出合作式取消信号 | cancel turn、关闭 leader |

读到 `Arc<Mutex<T>>` 时，不必立刻认为代码很复杂。先问三个问题：谁创建它，谁持有它，谁能修改它。读到 `mpsc::Sender` 时，顺着 receiver 找 actor 的主循环，通常比从类型定义向上猜更有效。

## 你会在这套仓库里看到的三种“消息”

不要把它们都叫 message：

1. **模型消息**：user、assistant、tool result 等，会进入 conversation。
2. **runtime 命令**：例如 `SessionCommand::Prompt`、`CompactSession`，控制 session actor 做事。
3. **跨进程协议消息**：ACP 或 leader protocol，在进程之间传输。

它们可能携带相似的文本，但生命周期、序列化方式和错误处理完全不同。后面看到 `prompt` 这个词时，先看它的类型，确认它属于哪一层。

## 第一个小实验

在源码 clone 里执行：

```bash
rg -n "pub enum SessionCommand|pub struct SamplerHandle|pub trait Tool|AgentSideConnection" crates
cargo metadata --no-deps --format-version 1 | jq '.packages | length'
```

第一个命令让你看到四个边界分别在哪里，第二个命令让你感受 workspace 的规模。你不需要立刻读懂这些定义；只要能把名词和文件位置连起来，下一章就有落脚点了。

## 把一次请求拆成四层

我自己读这类代码时，会把“用户输入了一个问题”拆成四个动作：

1. **意图层**：用户希望 Agent 做什么，可能来自 TUI 输入、headless 参数或 ACP 请求。
2. **编排层**：session 决定现在能不能开始一轮、要不要先压缩、是否需要权限确认。
3. **执行层**：sampler 请求模型，tool runtime 执行工具，workspace 提供文件和进程能力。
4. **展示层**：ACP notification、TUI event、headless JSON 或终端输出把结果交给外部观察者。

这四层的边界不是绝对的，但很适合定位问题。比如“模型没有看到某个工具”多半从 Agent/toolset 找；“工具已经返回但下一轮没有开始”要看 ChatState、turn 和 session；“终端卡住”则要把 TUI writer、ACP backpressure 和取消路径一起看。

## 同一个词为什么会有多个类型

`prompt` 可能是用户的一段文字、session command、模型请求中的 message，甚至是一个可以被重新排队的 interjection。`event` 可能是模型 delta、工具进度、ACP notification 或 UI redraw request。名字相似不说明它们可以互换，真正的答案在定义它的 crate 和发送它的 channel。

给自己留一张“类型翻译表”很有用：

| 我看到的词 | 先检查 | 不要直接推断 |
| --- | --- | --- |
| request | 谁构造、谁消费、是否可重试 | 它一定已经发到网络 |
| result | 面向模型、UI 还是调用方 | 用户看到的文本就是原始结果 |
| cancel | 谁发 token、谁观察 token | 所有子进程都已经退出 |
| state | 内存状态、历史记录还是 UI view model | 恢复后可以原样重建 |

## 我会画的第一张图

先不画完整架构，只画一条最小路径：

```text
user input
  -> session command
  -> request builder
  -> sampler request
  -> assistant/tool call
  -> tool result or final text
  -> session completion
  -> ACP/TUI output
```

每个箭头旁边都写上“同步调用、channel、文件记录还是跨进程 frame”。这样后面看到 `Arc`、`mpsc`、`watch` 或 JSON-RPC 时，我是在给已有的图补连接，而不是重新背一组术语。

如果你还不熟 Rust，先掌握三种阅读动作就够了：按定义跳转，按引用找调用方，按字段名找状态变化。不要先要求自己理解 lifetime、宏和所有 trait bound；它们重要，但不应该成为第一次理解业务链的门槛。
