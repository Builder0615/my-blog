# 35 Agent Mode 与 ACP 客户端集成

TUI 是 Grok Build 的一个客户端，Agent mode 则让其他编辑器、脚本或 SDK 通过 ACP 使用同一个 Agent runtime。用户入口是 [`15-agent-mode.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/15-agent-mode.md)，源码入口包括 `xai-grok-shell/src/agent/app.rs`、`xai-acp-lib/src/lib.rs`、`xai-grok-pager/src/acp` 和 `app/acp_handler`。

## 三种连接方式

| 模式 | transport | 适合 |
| --- | --- | --- |
| `grok agent stdio` | JSON-RPC over stdin/stdout | IDE、SDK、本地 child process |
| `grok agent serve` | WebSocket server | 多客户端或远端连接 |
| relay/leader 路径 | 本地复用/网络 relay | 复用 runtime、跨进程或管理场景 |

stdio 的 stdout 是协议通道，日志必须去 stderr；server mode 还要处理 bind address、secret、连接关闭和多个 client。

## ACP 初始化和 session 生命周期

一个最小客户端的时序是：

```text
spawn/connect
  -> initialize(protocolVersion, clientCapabilities)
  -> session/new(cwd, mcpServers, _meta)
  -> session/prompt(sessionId, prompt blocks)
  -> session/update notifications
  -> prompt response / turn completion
  -> session/load or disconnect/reconnect
```

`clientCapabilities` 会告诉 Agent client 是否提供 filesystem/terminal 等能力；`session/new` 的 cwd、MCP descriptors、rules、agentProfile、yoloMode、autoMode 等 `_meta` 会影响 session setup。能力声明不是装饰，Agent 可能根据它选择本地 tool 或发 extension request。

## `session/update` 是流式事件，而非最终字符串

用户指南列出的常见 `sessionUpdate` 包括 `agent_message_chunk`、`agent_thought_chunk`、`tool_call`、`tool_call_update` 和 `plan`。客户端应该按事件类型渲染/存储，不要把 thought、tool output 和 assistant text 拼成一段再猜边界。

典型客户端需要维护：

| 状态 | 作用 |
| --- | --- |
| current session id | 将后续 prompt 对准正确 session |
| request id map | 配对 response 与 request |
| tool call map | 合并 tool_call 和 tool_call_update |
| turn state | 判断 prompt 是进行中、完成、取消还是失败 |
| client capabilities | 决定 filesystem/terminal 请求的处理方式 |

## x.ai 扩展方法

基础 ACP 之上，Grok 还提供 `x.ai/` 扩展：filesystem、git、worktree、search、terminal、session、rewind、compact、auth、feedback、telemetry 等。它们把产品能力暴露给支持扩展的客户端，但扩展集合会随同步快照变化，初始化响应或方法发现应被当作运行时事实。

扩展 request 的失败不一定代表主 session 失败。例如 terminal 创建失败可能只让某个 bash tool 失败；worktree apply 失败要保留 child session；auth extension 可能改变后续 request 的 credential。

## 客户端 integration 的最小注意点

TypeScript 等 SDK 示例通常会 spawn `grok agent stdio`，用 line reader 读取 JSON-RPC，initialize 后 session/new，再在 prompt 期间消费 notification。真实代码不能只用一个 `once('line')` 处理所有 response，因为通知和 response 可以交错，应该按 JSON-RPC id/method 分流。

伪代码：

```text
on line:
  message = parse_json(line)
  if message.method == "session/update": route_notification(message)
  else if message.id exists: resolve_pending_request(message.id)
  else: report_protocol_error(message)
```

## reconnect 和 session/load

连接断开不等于 session 消失。客户端可以保存 session id，重连后调用 `session/load`；但正在进行的 turn、工具进程、pending permission 和 writer backpressure 不一定能原样恢复。客户端要识别 planned shutdown、relaunch、transport loss 和 session failure，采取不同策略。

## 本地验证

```bash
rg -n "AgentSideConnection|session/update|session/new|session/load|clientCapabilities|x.ai/|stdio|WebSocket" \
  crates/codegen/xai-grok-shell/src/agent \
  crates/codegen/xai-grok-pager/src/acp \
  crates/codegen/xai-grok-pager/src/app/acp_handler \
  crates/codegen/xai-acp-lib
cargo test -p xai-grok-pager acp
```

先用本地 fake client 验证消息路由，再接真实 IDE。这样能区分 ACP framing、capability、session setup 和模型请求错误。
