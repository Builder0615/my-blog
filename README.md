# Pi 学习文档

本套文档以 [Pi](https://github.com/earendil-works/pi)（earendil-works/pi）源码为蓝本，记录学习 Agent 开发的完整过程。

仓库是公开的，本项目不保存源码副本；所有源码引用均使用 GitHub HTTPS 链接。

## 学习顺序

建议按编号顺序阅读；每一章都建立在前一章的概念之上。

| 章节 | 文章 | 核心问题 | 源码包 |
| --- | --- | --- | --- |
| 01 | [总体架构与源码地图](posts/01-总体架构与源码地图/) | Pi 由哪些部分组成，代码应该从哪里读起 | monorepo |
| 02 | [模型与 Provider 抽象](posts/02-模型与Provider抽象/) | 一个 Agent 如何统一调用几十家 LLM | `pi-ai` |
| 03 | [Agent 核心循环](posts/03-Agent核心循环/) | Agent 的“思考-调用工具-再看结果”循环怎么实现 | `pi-agent-core` |
| 04 | [工具调用与参数校验](posts/04-工具调用与参数校验/) | 工具是什么、参数怎么校验、并行执行怎么处理 | `pi-agent-core` |
| 05 | [Harness 与会话持久化](posts/05-Harness与会话持久化/) | 会话、分支、存储如何支撑长任务 | `pi-agent-core` |
| 06 | [上下文管理与 Compaction](posts/06-上下文管理与Compaction/) | 上下文塞满后如何压缩、保留什么 | `pi-agent-core` |
| 07 | [Coding Agent 与 SDK](posts/07-Coding-Agent与SDK/) | 完整 CLI 如何复用核心循环并对外提供 SDK | `pi-coding-agent` |
| 08 | [TUI 与终端渲染](posts/08-TUI与终端渲染/) | 终端界面如何做到差分渲染、无闪烁 | `pi-tui` |
| 09 | [协议、存储与评测](posts/09-协议存储与评测/) | 跨进程协议、SQLite 存储、行为评测如何组织 | `pi-protocol` / `pi-storage` / `pi-evals` |
| 10 | [学习路线与动手实践](posts/10-学习路线与动手实践/) | 学完后如何做自己的 Agent 项目 | 综合 |
| 11 | [Q&A 学习记录](posts/11-Q&A学习记录/) | 学习过程中的二次提问记录 | - |

## 阅读方式

1. 先读 [01 章](posts/01-总体架构与源码地图/)，建立源码地图。
2. 每章先把“本章学习目标”记在心里，再对照“源码地图”打开文件。
3. 正文中的代码摘录都要回到 Pi 源码里看上下文，不要只看摘录。
4. 每个章节的“延伸问题”如果没看懂，去 [Q&A](posts/11-Q&A学习记录/) 追加一条记录，或者直接发起二次提问。

## 辅助阅读

- [Pi 官方文档入口](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/index.md)
- [pi-agent-core README](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md)
- [pi-ai README](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md)
- [pi-tui README](https://github.com/earendil-works/pi/blob/main/packages/tui/README.md)
- [pi-coding-agent SDK 文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md)

## 更新记录

每次新增或修改章节，在对应章节末尾的“延伸问题”或 `posts/11-Q&A学习记录/index.md` 中留下日期与说明。
