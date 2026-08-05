# 45 LSP、Codebase Graph 与 Remote

用户指南已经覆盖了很多直接可见的能力，但源码里还有三块容易被简化掉的支撑边界：LSP、代码库图和远程 workspace。它们都在帮助 Agent “理解代码并操作代码”，但算法、进程边界和网络边界完全不同。

本篇对应的入口包括 [05-configuration.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/05-configuration.md)、[15-agent-mode.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/15-agent-mode.md) 和 [17-sessions.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/17-sessions.md) 的相关段落。

源码地图：

- xai-grok-tools/src/implementations/lsp/{client,manager,dispatch,documents,diagnostics,workspace_open,restart}.rs；
- xai-codebase-graph/src/{index_manager,navigation,scope_graph,manager}.rs；
- xai-grok-shell/src/agent/mvp_agent/code_nav.rs、xai-grok-workspace/src/file_system/codebase_index.rs；
- xai-grok-workspace/src/workspace_ops.rs；
- xai-grok-workspace-client/src；
- xai-grok-workspace-types/src/rpc；
- xai-grok-shell/src/remote 和 xai-grok-shell/src/agent/relay.rs。

## LSP、Codebase Graph、grep 各自擅长什么

| 能力 | 主要问题 | 信息来源 | 典型失败 |
| --- | --- | --- | --- |
| grep/fuzzy search | 找包含某个文本的文件 | 文件内容 | 别名、宏、重命名和生成代码难处理 |
| LSP | 让语言服务器回答定义、引用、诊断、格式化 | 编译器/语言服务器 | server 未安装、初始化失败、版本不兼容 |
| Codebase Graph | 从源码解析符号、作用域和引用关系 | tree-sitter/index/cache | 语言覆盖、语法错误、增量索引滞后 |
| remote workspace | 把文件、Git、code nav、hunk 等操作路由到远端 | typed RPC / WebSocket | 网络断线、权限和远端版本不一致 |

这不是“图比 grep 高级”这么简单。grep 适合快速发现；LSP 适合复用语言服务器已有的语义；Codebase Graph 适合本地、可缓存、可增量的导航；remote workspace 解决的是操作位置，而不是搜索算法。

## Codebase Graph 的增量模型

源码文档说明 xai-codebase-graph 提供初始索引、增量 reindex、并行处理、memory-mapped I/O、go-to-definition 和 go-to-references。它还有 IndexManagerHandle，文件事件进入 manager，查询可以直接对 manager 发命令，避免每次复制完整 index。

源码摘录：[xai-codebase-graph/src/lib.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-codebase-graph/src/lib.rs) 的示例：

~~~rust
let handle = IndexManager::spawn(config);
handle.send_event(FileEvent::modified("src/main.rs".into()))?;
let result = handle.goto_definition_blocking(file, 10, 15)??;
~~~

这个设计把“文件变化”和“查询”放进同一个 manager 边界，避免一个线程读旧 snapshot、另一个线程写新索引时互相踩踏。代价是多一个 actor/channel、缓存一致性和锁；索引不可能比最新文件事件更快。

## LSP 的生命周期

LSP client 也不是每次调用都启动一个命令：

~~~text
读取项目 LSP 配置
  -> 启动或复用 language server
  -> initialize / workspace folders
  -> open/change 文档同步
  -> request definition/references/diagnostics
  -> 处理 pending、超时、restart
  -> session/workspace 结束时清理
~~~

Manager、pending request、documents、refresh 和 restart 模块分别对应这些状态。Agent 使用 LSP 时，还要考虑文件是否已经通过 hunk tracker 修改但尚未写入、远端 client 是否持有文件内容，以及 language server 的根目录是否与 Grok workspace root 相同。

## LSP 和 Codebase Graph 怎样协同

~~~mermaid
flowchart TD
    F["文件读写 / editor change"] --> E["file event"]
    E --> G["Codebase Graph incremental index"]
    E --> L["LSP didOpen/didChange"]
    Q["Agent code navigation"] --> D{"选择后端"}
    D -->|语言服务器可用| LQ["LSP request"]
    D -->|本地 graph 可用| GQ["Graph query"]
    D -->|都不可用| SQ["search/fuzzy fallback"]
    LQ --> R["locations / diagnostics"]
    GQ --> R
    SQ --> R
    R --> A["tool result -> model"]
~~~

图依据 lsp manager/dispatch、codebase graph index_manager/navigation 和 tools 的 code navigation 接线。它表达的是后端选择关系，实际优先级、语言覆盖和 fallback 条件需要继续看 code_nav 与 config。

选择两个后端的收益是可用性和速度：LSP 可能提供更准确的语义，graph 可以在没有完整 language server 的项目里快速工作；代价是两套索引的结果可能短时间不一致，诊断中必须注明来源。

## Remote Workspace 的类型安全思路

源码文档对 WorkspaceOps 直接区分 Local 与 Proxy：

~~~text
Local:
  tool call -> WorkspaceHandle / local FinalizedToolset

Proxy:
  tool call -> typed WorkspaceRpc -> WebSocket hub -> remote workspace server
~~~

每个 RPC request struct 携带 METHOD，并实现 Serialize、Deserialize 和 WorkspaceRpc；proxy client 和 server dispatch 共用同一个 request type。新增字段或改 response 时，编译器可以帮忙暴露两端不一致。

这比“把任意 JSON 发到远端”更啰嗦，但收益很大：

- code navigation、文件、Git、hunk、worktree、skills 和 session RPC 有清晰名字；
- 本地和远端能复用同一套业务类型；
- 相对容易测试序列化、权限和错误映射；
- 远端 transport 失败可以分类为可重试或致命。

代价是 wire schema 变成长期兼容负担；本地调用和远程调用也可能有不同延迟、权限和文件观察时机。

## Remote、Relay、ACP 不要互相替代

| 层 | 负责 |
| --- | --- |
| ACP | Agent 与外部客户端之间的 session/prompt/update 协议 |
| Workspace RPC | Agent 运行时访问远端文件、Git、导航和工作区能力 |
| Relay | 在 WebSocket/网络层维持 Agent 或 leader 的连接、重连和认证 |
| Remote settings | 从服务端同步模型、开关、策略或组织配置 |

一次远程运行可能同时经过 ACP client、Agent relay、Workspace proxy 和 MCP HTTP；出错时要知道是哪一条链路断了。

## 远程请求的时序

~~~mermaid
sequenceDiagram
    participant C as "ACP client"
    participant A as "Agent runtime"
    participant W as "WorkspaceOps"
    participant H as "Hub/WebSocket"
    participant R as "Remote workspace"
    C->>A: session/prompt
    A->>W: code nav / read / git operation
    W->>W: choose Local or Proxy
    W->>H: typed WorkspaceRpc
    H->>R: dispatch METHOD
    R-->>H: response or WorkspaceError
    H-->>W: typed result
    W-->>A: tool output
    A-->>C: session/update
    H--xW: transport loss
    W->>W: classify fatal/retry
~~~

图的依据是 WorkspaceOps 的 Local/Proxy 文档、WorkspaceRpc 类型、workspace-client transport 和 shell relay。它没有把认证 refresh、重连 backoff 和 session resume 的所有分支画出来；这些要看 remote/client.rs 和 relay.rs。

## 失败路径

| 现象 | 判断方向 |
| --- | --- |
| 定义跳转结果为空 | LSP 未初始化、graph 未索引、位置坐标、语言不支持 |
| graph 看不到刚改的文件 | FileEvent 未送达、增量 index pending、缓存锁 |
| LSP 与 graph 结果不同 | 两个后端观察到的版本/根目录不同 |
| 远程读文件超时 | WebSocket、hub、远端文件系统或 backpressure |
| Git 操作路径错误 | RPC 要求相对 workspace root，不能直接传任意绝对路径 |
| relay 断线 | token、网络、liveness、重连和 session 是否仍可用 |
| remote policy 覆盖本地配置 | 要区分 user config、remote settings 和 policy pin |

## 外部资料：如何把网上文章放进学习路径

我会把外部材料分成三类，并在正文中标注角色：

1. 协议原始资料：xAI 的 [Grok Build overview](https://docs.x.ai/build/overview)、ACP 官方 [Architecture](https://agentclientprotocol.com/get-started/architecture)、MCP 官方 [Architecture](https://modelcontextprotocol.io/docs/learn/architecture)。它们适合解释协议的公共概念和产品入口，不替代仓库快照。
2. 实践型资料：例如 [ACP explained for coding agents](https://learncodecamp.net/agent-client-protocol-acp-explained/)，适合把 JSON-RPC、client/server、LSP 类比说得更直观；其中关于 Grok Build 当前源码的判断仍要回到本仓库。
3. 社区经验：例如 Reddit 上的 [Grok Build 开源架构讨论](https://www.reddit.com/r/HowToAIAgent/comments/1uxr891/xai_open_sourced_grok_build_13m_lines_of_rust_i/) 和 [配置实践整理](https://www.reddit.com/r/grok/comments/1tnrbmv/i_collected_practical_grok_build_setup_patterns/)。它们可能提供安装坑、使用方式或个人推断，但不是官方承诺，也不作为源码事实的证据。

记录外部链接时要保留访问日期和“官方/第三方/社区”标签。本次外部资料访问日期为 2026-08-05；当前笔记使用的仓库源码快照仍是 ed6d543。网页内容更新后，不能自动推翻本地源码判断。

## 本地验证

~~~bash
rg -n "LspBackend|IndexManager|IndexManagerHandle|goto_definition|goto_references|WorkspaceOps|WorkspaceRpc|Local|Proxy|relay|is_transport_fatal" \
  crates/codegen/xai-grok-tools/src/implementations/lsp \
  crates/codegen/xai-codebase-graph/src \
  crates/codegen/xai-grok-workspace/src \
  crates/codegen/xai-grok-workspace-client/src \
  crates/codegen/xai-grok-shell/src/remote \
  crates/codegen/xai-grok-shell/src/agent/relay.rs

cargo test -p xai-codebase-graph
cargo test -p xai-grok-tools lsp
cargo test -p xai-grok-workspace
~~~

阅读这部分时我会为每次导航结果标记后端、文件版本和 workspace 根目录。这样遇到“模型跳到了错误定义”时，能判断是算法问题、索引延迟还是远程文件版本不一致。
