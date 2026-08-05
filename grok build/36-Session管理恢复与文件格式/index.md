# 36 Session 管理、恢复与文件格式

我把 Session 理解成“一个可以被重新接上的 Agent 工作现场”。它不只是聊天记录：还要记住当前目录、模型、规则、计划、压缩点、回退点、后台任务和客户端是否正在等待某个请求。对应的产品入口是 [17-sessions.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/17-sessions.md)；源码主要分布在 xai-grok-shell/src/session 和 pager 的 session dispatch。

## Session、Turn、Prompt 不要混成一个词

| 名称 | 白话含义 | 生命周期 |
| --- | --- | --- |
| Session | 一次可恢复的工作现场 | 可以跨进程、跨天继续 |
| Prompt | 用户或系统送入的一次请求 | 可能排队，也可能被取消 |
| Turn | Agent 从收到 prompt 到产生完成/取消结果的一轮 | 可能包含多次模型采样和工具调用 |
| Conversation item | 写入历史的一个结构化消息 | 由 ChatState 和持久化层共同维护 |

这样分层是有实际原因的。用户按下 Esc 取消的是当前 turn，不应该把整个 session 删除；/resume 恢复的是 session，不意味着上一个正在运行的进程还能继续；/fork 复制的是历史起点，后续写入应该与父 session 分开。

## 磁盘上的目录不是“一个大 JSON”

用户指南给出的默认位置是 ~/.grok/sessions/<encoded-cwd>/<session-id>/。公开快照中可以看到多种相互配合的文件：

| 文件 | 用途 | 为什么不合并 |
| --- | --- | --- |
| summary.json | session 基本信息、标题、cwd、模型等 | picker 和 dashboard 不必扫描完整对话 |
| updates.jsonl | session update / 事件流 | 适合追加和逐条恢复 |
| chat_history.jsonl | 模型上下文所需的对话项 | 与 UI 事件解耦 |
| plan.json | 计划状态和审批相关信息 | 恢复时能重新建立审批等待 |
| rewind_points.jsonl | 文件状态、turn 边界和回退点 | 回退不是简单删消息 |
| signals.json | 取消、关闭、重新加载等协作信号 | 让外部客户端可以观察状态 |
| feedback.jsonl | 用户反馈事件 | 不污染模型可见历史 |

这些文件的具体字段会随快照变化，不能把上表当成稳定公共 API。它更适合帮助我定位读写入口：session storage 负责文件，SessionActor 负责顺序，pager 负责用户动作和展示。

## 持久化为什么通过 channel 进入 actor

源码摘录：[session/chat_persistence.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/session/chat_persistence.rs) 中的生产实现把 ChatPersistence 转换成 PersistenceMsg：

~~~rust
pub(crate) struct ChannelChatPersistence {
    tx: mpsc::UnboundedSender<PersistenceMsg>,
}

impl ChatPersistence for ChannelChatPersistence {
    fn persist_message(&mut self, item: &ConversationItem) {
        let _ = self.tx.send(PersistenceMsg::Chat(item.clone()));
    }

    fn flush(&mut self) {
        let _ = self.tx.send(PersistenceMsg::Flush);
    }
}
~~~

这里的重点不在 mpsc 这个词，而在“写历史的顺序由一个接收方决定”。模型流、工具回调、UI 更新都可能同时发生；如果它们各自直接打开同一个 JSONL 文件，顺序、部分写入和 flush 时机都会变得很难推理。

选择 channel + actor 的收益：

- 上层只发送 typed message，不需要知道文件句柄；
- 所有 append、replace、flush 在一个顺序化边界内执行；
- persist_working_directory_switch_and_ack 可以用 oneshot 回报“这条切换记录已经落盘”；
- session actor 关闭时可以等待写入收尾，而不是让 UI 自己猜。

代价也很明确：channel 断开后，上层只能得到 BrokenPipe 或 indeterminate 状态；写入失败不是模型错误，诊断时要单独看 persistence 日志。

## 恢复一轮请求的时序

~~~mermaid
sequenceDiagram
    participant U as 用户或客户端
    participant P as Pager dispatch
    participant A as SessionActor
    participant S as Session storage
    participant C as ChatState
    U->>P: /resume 或 session/load
    P->>S: 读取 summary、history、updates
    S-->>P: 恢复材料与版本信息
    P->>A: 建立 actor + channel
    A->>C: replay chat history
    A->>A: 恢复 plan、rewind、signals
    A-->>P: idle/awaiting approval/failed
    U->>P: 新 prompt
    P->>A: SessionCommand::Prompt
    A->>S: append prompt/turn events
    A-->>U: session/update + response
~~~

图的源码依据是 xai-grok-pager/src/app/dispatch/session/load.rs、lifecycle.rs、fork.rs 与 xai-grok-shell/src/session 的 command/persistence 模块。它能说明组件关系，不能证明所有文件都在同一个线程中读完；具体调度仍要看对应 async 函数和测试。

## resume、continue、fork、rewind 的差别

| 操作 | 它改变什么 | 容易误解的地方 |
| --- | --- | --- |
| /resume <id> | 打开指定 session | 不是恢复旧进程内的 tokio task |
| -c / continue | 选择当前目录最近可继续的 session | “最近”有目录、时间和可加载条件 |
| /fork | 从某个历史点创建新的工作现场 | 父 session 后续追加不会自动同步到子 session |
| /rewind | 回到记录过的文件/turn 状态 | 不是只截断 chat_history |
| /compact | 生成更短的上下文并保留继续工作的入口 | 压缩后的事实可能改变模型看到的文本 |
| /session-info | 查询元数据、路径和统计 | 诊断命令本身不等于完整导出 |

## 为什么回退需要 rewind point

如果只删掉末尾几条 assistant 消息，工作区里的文件、shell 进程、hunk tracker 和模型上下文会彼此不一致。Grok Build 把回退点单独记录，再由 workspace/session 的文件状态、hunk 和 chat history 协作处理。这个设计多了一套状态，却避免“聊天回去了，文件没回去”的假成功。

一个适合小白的心智模型是：

~~~text
chat history      = Agent 记得说过什么
rewind point      = 当时文件大致长什么样
workspace hunk    = 哪些改动来自哪个 turn
session metadata  = 这个现场在哪里、用什么配置打开
~~~

## 失败路径要逐类看

| 现象 | 可能位置 | 应该检查 |
| --- | --- | --- |
| session picker 看得到但加载失败 | JSONL 某行损坏、版本不兼容 | summary.json、解析日志、具体文件 |
| prompt 已显示但 resume 后消失 | chat append 尚未 flush 或写入 actor 已退出 | persistence channel、flush ack |
| cwd 被删除 | 工作区恢复失败 | session cwd、worktree 快照、fallback cwd |
| plan approval 卡住 | 文件显示 awaiting，但 live waiter 不在 | RestorePlanApproval 和 ACP reverse request |
| fork 后改动串到父目录 | worktree 创建失败后回退到共享 workspace | worktree 日志、权限和 git 状态 |
| 取消后仍有任务输出 | cancel 只结束 turn，后台任务仍被允许存活 | CancelOptions、task coordinator 和 notification drain |

这里的“可恢复”是有边界的：静态历史通常能恢复，运行中的外部进程、网络连接和交互式权限请求通常只能重新建立。

## 本地阅读与验证

~~~bash
rg -n "PersistenceMsg|ChannelChatPersistence|replace_history|AppendCwdSwitchAndAck|SessionCommand|PromptCompletionKind" \
  crates/codegen/xai-grok-shell/src/session

rg -n "dispatch_(load|new|fork)|rewind|session/load|session_info" \
  crates/codegen/xai-grok-pager/src/app/dispatch \
  crates/codegen/xai-grok-pager/src/slash

cargo test -p xai-grok-shell session
cargo test -p xai-grok-pager session
~~~

我会把一次恢复失败拆成“文件读取、结构解析、actor 初始化、模型重新请求”四段记录。只说“session 坏了”对定位帮助很小。
