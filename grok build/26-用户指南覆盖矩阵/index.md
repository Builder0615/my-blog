# 26 用户指南覆盖矩阵

前面的 00—25 主要按源码运行链组织；这一组补上产品用户指南里的具体功能。用户指南和源码不是同一种文档：用户指南告诉我“产品承诺什么”，源码告诉我“当前快照怎样实现”，测试告诉我“哪些路径真的被验证”。我会把三者分开对照。

本矩阵按公开仓库提交 [`ed6d543`](https://github.com/xai-org/grok-build/commit/ed6d543643628663873c5de28298e022ed634238) 建立。表里的“深挖单元”不是 Q&A；它们是正文，负责把功能讲清楚。Q&A 只保留你实际提出的问题。

## 24 篇 user guide 现在落到哪里

| 用户指南 | 学习单元 | 重点不再只停留在什么名字 |
| --- | --- | --- |
| [`01-getting-started.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/01-getting-started.md) | [27 第一次运行与基本交互](../27-第一次运行与基本交互/) | 安装、TUI 两区、`@` 引用、首次 turn、常用 CLI |
| [`02-authentication.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/02-authentication.md) | [31 认证全流程与凭据刷新](../31-认证全流程与凭据刷新/) | OAuth、API key、OIDC、external provider 的 stdout/stderr 合约 |
| [`03-keyboard-shortcuts.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/03-keyboard-shortcuts.md) | [28 键盘输入与快捷键状态机](../28-键盘输入与快捷键状态机/) | focus、Esc、vim/simple、问题卡片与输入优先级 |
| [`04-slash-commands.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/04-slash-commands.md) | [29 Slash Commands 与命令分发](../29-Slash%20Commands与命令分发/) | registry、别名、参数、模式限制和 skill 命令 |
| [`05-configuration.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/05-configuration.md) | [33 配置系统与项目规则](../33-配置系统与项目规则/) | `config.toml`、`pager.toml`、环境变量、项目层与 LSP 配置 |
| [`06-theming.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/06-theming.md) | [30 TUI 主题终端外观与配置](../30-TUI主题终端外观与配置/) | 主题、auto、颜色量化、minimal 与 pager 配置 |
| [`07-mcp-servers.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/07-mcp-servers.md) | [39 MCP Server 完整生命周期](../39-MCP%20Server完整生命周期/) | stdio/HTTP/SSE、OAuth、命名、发现、重启与子代理继承 |
| [`08-skills.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/08-skills.md) | [40 Skills、Plugins、Hooks 扩展机制](../40-Skills、Plugins、Hooks扩展机制/) | frontmatter、scope、自动调用、slash 命令与 reload |
| [`09-plugins.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/09-plugins.md) | [40 Skills、Plugins、Hooks 扩展机制](../40-Skills、Plugins、Hooks扩展机制/) | marketplace、信任、安装、启用和组织限制 |
| [`10-hooks.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/10-hooks.md) | [40 Skills、Plugins、Hooks 扩展机制](../40-Skills、Plugins、Hooks扩展机制/) | JSON、事件、阻塞输出、退出码、HTTP hook 与安全 guard |
| [`11-custom-models.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/11-custom-models.md) | [32 自定义模型与 Provider 协议](../32-自定义模型与Provider协议/) | 三种 API backend、凭据优先级、context、headers 和 endpoint |
| [`12-project-rules.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/12-project-rules.md) | [33 配置系统与项目规则](../33-配置系统与项目规则/) | 文件名、目录层级、优先级、gitignore 与 `.grok/` |
| [`13-memory.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/13-memory.md) | [41 Memory 搜索、注入与 Dream](../41-Memory搜索注入与Dream/) | hybrid scoring、MMR、衰减、flush、dream、staleness |
| [`14-headless-mode.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/14-headless-mode.md) | [34 Headless 脚本与输出协议](../34-Headless脚本与输出协议/) | 四种输出格式、usage、session、CI 与退出码 |
| [`15-agent-mode.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/15-agent-mode.md) | [35 Agent Mode 与 ACP 客户端集成](../35-Agent%20Mode与ACP客户端集成/) | stdio、server、relay、extension methods 与 SDK |
| [`16-subagents.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/16-subagents.md) | [37 Subagent、Persona 与隔离](../37-Subagent、Persona与隔离/) | agent/persona、输入输出契约、capability、worktree、深度限制 |
| [`17-sessions.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/17-sessions.md) | [36 Session 管理、恢复与文件格式](../36-Session管理恢复与文件格式/) | JSONL、summary、rewind、fork、resume、SQLite 搜索 |
| [`18-sandbox.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/18-sandbox.md) | [42 Sandbox 与 Permission 实战](../42-Sandbox与Permission实战/) | profile、custom deny、平台、环境变量和 resume 固定性 |
| [`19-plan-mode.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/19-plan-mode.md) | [38 Plan、Background Task 与 Workflow](../38-Plan、Background%20Task与Workflow/) | plan file、approval、反馈、编辑门禁与 compaction |
| [`20-background-tasks.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/20-background-tasks.md) | [38 Plan、Background Task 与 Workflow](../38-Plan、Background%20Task与Workflow/) | background、monitor、loop、scheduler、tasks pane |
| [`21-terminal-support.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/21-terminal-support.md) | [43 Terminal Support、Voice 与媒体](../43-Terminal%20Support、Voice与媒体/) | color、clipboard、tmux、Zellij、VS Code、voice |
| [`22-permissions-and-safety.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/22-permissions-and-safety.md) | [42 Sandbox 与 Permission 实战](../42-Sandbox与Permission实战/) | mode、rule matching、interactive grants、hook 与 sandbox |
| [`23-dashboard.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/23-dashboard.md) | [44 Dashboard、Usage 与 OTEL 监控](../44-Dashboard、Usage与OTEL监控/) | roster、peek、focus、搜索过滤和持久化 |
| [`24-monitoring-usage.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/24-monitoring-usage.md) | [44 Dashboard、Usage 与 OTEL 监控](../44-Dashboard、Usage与OTEL监控/) | OTLP、metrics、events、脱敏和 collector |

## 还要回到源码的支撑边界

用户指南覆盖了产品入口，但源码中还有一些不适合塞进用户指南的边界：

| 支撑能力 | 代表位置 | 本套笔记的处理 |
| --- | --- | --- |
| LSP、code navigation | `xai-grok-tools/src/implementations/lsp`、`agent/mvp_agent/code_nav.rs` | 在 15 和 45 分别讲工具协议与产品接线 |
| 图片、PDF、PPTX | `xai-grok-tools/src/implementations/read_file/{image,pdf,pptx}.rs` | 在 14、43 说明输入、抽取和媒体渲染边界 |
| codebase graph | `xai-codebase-graph`、`xai-grok-workspace/src/file_system/codebase_index.rs`、`xai-grok-shell/src/agent/mvp_agent/code_nav.rs` | 在 45 说明索引与代码导航，不把它误写成 grep |
| remote、relay、workspace client | `xai-grok-shell/src/remote`、`relay`、`xai-grok-workspace-client` | 在 06、17、35、45 拆本地/远端边界 |
| voice、notifications、feedback | pager `voice`、`notifications`、shell extensions | 在 23、43、44 说明产品服务与失败路径 |

这张表是“覆盖审计”，不是声称每个文件都已逐行讲完。下一次上游同步时，我会重新对照 user guide 文件名、package 数量和这些支撑目录。
