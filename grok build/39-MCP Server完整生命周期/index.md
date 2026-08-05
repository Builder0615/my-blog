# 39 MCP Server 完整生命周期

MCP 把外部工具、资源和 prompt 以标准协议接进 Agent。Grok Build 的用户入口是 [07-mcp-servers.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/07-mcp-servers.md)，实现集中在 xai-grok-mcp，再由 session 和 tool bridge 接入。

源码入口：

- [xai-grok-mcp/src/lib.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-mcp/src/lib.rs)：crate 边界、credentials、OAuth、servers、wire；
- [xai-grok-mcp/src/servers.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-mcp/src/servers.rs)：transport、client lifecycle、tool invocation、diff refresh；
- xai-grok-mcp/src/credentials.rs、oauth.rs、liveness.rs：凭据、浏览器授权、存活探测；
- xai-grok-shell/src/session/acp_session_impl/mcp.rs、mcp_snapshot.rs：session 接入和快照；
- xai-grok-tools/src/bridge.rs：把远端 MCP tool 放进本地 dispatch。

## MCP Server 不是一个静态配置项

配置只是描述 server；运行时还要经历：

~~~text
配置发现
  -> 名称/URL/transport 校验
  -> spawn child 或建立 HTTP/SSE 连接
  -> initialize 握手
  -> list tools/resources/prompts
  -> 规范化名称并注册到 ToolBridge
  -> session prompt 可见
  -> health/liveness、配置 diff、重连或 teardown
~~~

Server 的 tool 名通常会被限定为 server__tool，防止两个 server 暴露同名工具时互相覆盖。servers.rs 还提供名称校验和 descriptor segment 清理；这不仅是美观问题，也关系到 tool 文件和 prompt 路径的一致性。

## 初始化状态为什么用 enum

源码摘录：[servers.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-mcp/src/servers.rs) 把初始化进度建模为 NotStarted、Starting、Finished，并在 Finished 后允许后台握手继续排空：

~~~rust
pub enum InitProgress {
    NotStarted,
    Starting {
        handshaking: std::collections::HashSet<McpServerName>,
    },
    Finished {
        handshaking: std::collections::HashSet<McpServerName>,
    },
}
~~~

如果只用 initialized: bool 和 initializing: bool，会出现两者同时为 true、或者表面已完成但某个 server 仍握手中的组合。enum 让非法组合更难出现，也让 match 分支必须处理“正在启动、已放行主流程但后台仍在收尾”。

## 完整时序

~~~mermaid
sequenceDiagram
    participant C as Config/Remote settings
    participant M as MCP manager
    participant T as stdio / HTTP transport
    participant R as MCP server
    participant B as ToolBridge
    participant A as SessionActor
    C->>M: server descriptors
    M->>M: validate name + normalize URL
    M->>T: spawn child or connect
    T->>R: initialize + capabilities
    R-->>T: tools/resources/prompts
    T-->>M: client ready
    M->>B: register qualified tools
    M-->>A: init progress / notifications
    A->>B: dispatch server__tool
    B->>R: call_tool
    R-->>B: result or protocol error
    C-->>M: config diff
    M->>M: retain unchanged, teardown removed, restart changed
~~~

图依据 InitProgress、McpConfigDiff、servers.rs 的 transport 和 tool registration 代码，以及 session 的 MCP 接入。它没有把 MCP server 内部实现画出来；外部 server 的能力、延迟和错误仍是黑盒。

## 为什么初始化不能阻塞所有请求

某个远程 server 可能慢、需要 OAuth 或暂时不可达，但用户仍可能只想读本地文件。快照中的设计让 finish_init 可以先释放非 MCP 工作，单个 server 的 handshake 在后台继续。收益是首屏和非 MCP 请求更快；代价是 tool 列表在短时间内变化，模型第一次 prompt 可能看不到尚未 ready 的 server。

我会把“可用”分成三档：

| 状态 | Agent 能做什么 |
| --- | --- |
| descriptor 已读 | 显示配置和诊断信息 |
| client ready | 注册并调用 MCP tool |
| handshake pending | 等待、重试或走非 MCP 工具 |

## OAuth、凭据和 transport 的边界

stdio server 的凭据通常进入子进程环境或启动参数，HTTP server 的凭据则由 OAuth store、header 或 transport 配置提供。mcp_credentials.json 和普通模型 API key 不应该混用。HTTP 的重连还可能需要 backoff、liveness 和 URL 规范化。

MCP 官方架构页把协议分成 JSON-RPC data layer 与 transport layer，并说明 client/server 的 lifecycle、capabilities、tools、resources、prompts 和 notifications。它适合作为协议背景：[MCP Architecture](https://modelcontextprotocol.io/docs/learn/architecture)。这不是 Grok Build 的源码证明；Grok 的具体状态机仍以 xai-grok-mcp 快照为准。

## 失败路径

| 现象 | 实现层需要处理什么 |
| --- | --- |
| tool 名含空格或超长 | 在注册前拒绝或清理，不能让 provider API 在晚些时候拒绝 |
| stdio child 退出 | 标记 server unavailable，回收 process scope，避免僵尸工具 |
| HTTP 401 或过期 OAuth | 进入 OAuth refresh/重新授权，不要把错误伪装成 tool 空结果 |
| server 初始化很慢 | 保持 pending，非 MCP 请求继续；必要时给用户状态 |
| 配置修改 | diff 出 added、removed、retained，避免无变化的 server 全部重启 |
| 子代理继承 | 继承的是经过策略解析的 MCP 快照，不等于无条件复制父连接 |

## 本地验证

~~~bash
rg -n "InitProgress|McpConfigDiff|validate_tool_name|sanitize_descriptor_segment|register|handshake|liveness|oauth" \
  crates/codegen/xai-grok-mcp \
  crates/codegen/xai-grok-shell/src/session/acp_session_impl \
  crates/codegen/xai-grok-tools/src/bridge.rs

cargo test -p xai-grok-mcp
cargo test -p xai-grok-shell mcp
~~~

可以先用一个假的 stdio MCP server 验证 framing、初始化和 tool name，再测试 OAuth；否则网络问题会把协议问题遮住。

