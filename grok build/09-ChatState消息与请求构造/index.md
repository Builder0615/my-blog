# 09 ChatState、消息与请求构造

模型每次调用都需要一份上下文。初学者很容易以为上下文就是一个 `Vec<Message>`，但生产 Agent 还要处理工具结果、压缩段、token usage、system prompt 变更、历史持久化和并发 command。`xai-chat-state` 就是把这组状态单独拿出来的地方。

## ChatState 是谁的状态

[`xai-chat-state/src/lib.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-chat-state/src/lib.rs) 的模块结构包含 actor、commands、request_builder、persistence、compaction_mode 和 compaction_utils。公开 handle 把命令发给 actor，actor 拥有真正的 conversation 和状态。

可以先把它和 session 分开：

| 层 | 保存什么 | 更关心什么 |
| --- | --- | --- |
| ChatState | 模型上下文、消息、usage、request 相关状态 | 下一次请求应该带什么 |
| SessionActor | prompt、工具、权限、压缩、完成状态 | 这一轮产品行为如何继续 |
| Session persistence | 可恢复的记录 | 进程退出后怎样重建 |
| TUI/AppView | 可视化状态和输入 | 用户现在看到什么 |

它们会互相通知，但不能用其中一层替代另一层。

## 三种消息不要混

`ConversationItem`、ACP message 和 UI event 都可能包含文字，但用途不同：

1. conversation item 会进入模型 request；
2. ACP message 在客户端和 Agent 之间传递操作与结果；
3. UI event 只表达显示、交互或进度。

工具调用的完整生命周期通常是“assistant item 中的 call → 工具执行 → tool result item → 下一次 request”。session 还会把这组内容记录到可恢复历史中。看到 `record_response`、`record_tool_result` 或 `session update` 时，先问它更新的是哪一层。

## request builder 在哪里

一次 request 通常不是直接从 session 的数组序列化，而是由 ChatState 的 request builder 结合：

- conversation/history；
- system prompt；
- 当前有效工具定义；
- model config 和 reasoning effort；
- hosted tools、memory reminder、trace context；
- token budget、session id、turn index 和 deployment metadata。

第 08 章已经说明 Agent 定义如何提供 prompt 和 tools；这里要继续看 [`turn.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/session/acp_session_impl/turn.rs) 里的调用方，以及 ChatState 的 request builder 测试。

## actor handle 的价值

如果 TUI、headless、leader 和 session 内部都可以直接修改同一个 conversation，消息顺序会很难保证。handle 把修改变成类型化 command，actor 串行处理，外部只能等待结果或订阅事件。

这种设计也有代价：你不能只跳转到一个字段赋值就理解状态变化，要找发送 command 的调用方、actor 的 match 分支和持久化更新。对于小白，最有效的笔记格式是三列：`command`、`状态变化`、`对外事件`。

## 压缩怎样改变 ChatState

compaction 不一定删除原始历史。它会生成 segment/checkpoint 或 summary，再由 request builder 选择“摘要 + 保留尾部”作为下一次模型上下文。session persistence 还可能保留更完整的记录，以便恢复、rewind 或诊断。

这就是“历史存在”和“历史进入模型上下文”两个概念。第 19 章会具体追阈值和压缩算法。

## ChatState 需要同时服务三个观察者

同一段对话状态通常要给三类观察者使用：

- **模型观察者**：关心符合 provider schema 的 system、user、assistant、tool 内容和 token/usage。
- **session 观察者**：关心可恢复的 entry、completion、compaction、branch/rewind 和错误原因。
- **用户界面观察者**：关心流式 delta、工具进度、权限提示、标题、状态和可渲染 block。

这三者不应共享一份“显示文本”作为唯一真相。UI 为了速度可能先显示 delta，模型请求需要结构化 tool call，session 则要在结算时写入可恢复记录。读 `ChatState` 时看到一个 `record_*` 函数，最好继续找它的调用方和对外 event。

## request builder 的职责边界

request builder 不是简单把 vector 拼起来。它可能要处理：

1. 选择当前 model 和 endpoint 需要的消息格式。
2. 把 Agent definition 的 system/tool context 放到 provider 能理解的位置。
3. 从 ChatState 选择可见历史，考虑 compaction、branch、reminder 和 interjection。
4. 计算或附带 token budget、tool choice、response format、streaming 和 metadata。
5. 在不满足 context window、参数非法或历史损坏时返回明确错误。

这也是为什么“模型没看到上一条工具结果”不能只查 UI。要比较三份东西：session 历史里有什么、request builder 选择了什么、实际 sampler request 序列化了什么。

## 一个适合小白的排查例子

假设用户看到工具执行成功，但模型下一条回复像没执行过：

```text
UI event 有工具完成吗？
  -> 有：查 tool result 是否 record 到 ChatState
  -> 有：查 request builder 是否选中这条 history
  -> 有：查 provider adapter 是否丢弃/改写 tool message
  -> 有：查模型响应是否来自旧 request 或重试 request
```

这条排查链把“状态没写入”“上下文没选中”“协议转换丢失”“并发重试错位”分开了。每一层都可以用日志、单元测试或 snapshot request 验证。

## 小实验

```bash
rg -n "ConversationItem|build_request|token|usage|record_.*(response|tool)|ChatState" \
  crates/codegen/xai-chat-state \
  crates/codegen/xai-grok-shell/src/session
```

选择一条 `SessionCommand::Prompt` 路径，写出 request builder 读取的字段，并标注每个字段来自 Agent、ChatState、workspace 还是 CLI。能完成这个表，就不会把 system prompt、conversation 和 UI transcript 混为一件事。
