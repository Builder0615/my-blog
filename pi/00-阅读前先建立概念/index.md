# 00 阅读前先建立概念

Pi 这套源码更适合拿来建立 Agent 的第一张图，但它仍然不是“只要会 JavaScript 就能直接读”的项目。这里先把后面经常出现的词解释清楚。

## Pi 的四个核心包

| 包 | 白话职责 |
| --- | --- |
| `pi-ai` | 把不同模型/Provider 的输入输出统一成类型和事件流 |
| `pi-agent-core` | 处理 assistant response、tool call、tool result 和继续循环 |
| `pi-coding-agent` | 把 Agent、session、工具、扩展、CLI、TUI 和 SDK 组装成产品 |
| `pi-tui` | 终端输入、组件、差分渲染和屏幕输出 |

当前源码还包含 `pi-protocol`、`pi-client`、`pi-server`、storage、evals 和多个 coding-agent 子目录。它不是只有一个 `agent-loop.ts`。

## 先记住三层消息

1. `pi-ai` 的 `Message` 是模型协议能理解的 user/assistant/toolResult 等消息。
2. `pi-agent-core` 的 `AgentMessage` 可以包含 Agent 自己的扩展消息和工具状态，到了模型边界才通过 `convertToLlm` 转换。
3. `pi-coding-agent` 的 `AgentSessionEvent` 是给 TUI、RPC、扩展和持久化使用的产品事件。

这也是旧笔记里“消息只有三种角色”需要补充的地方：三角色适合描述 LLM 边界，但不能概括 Agent runtime 内部的所有消息。

## 当前源码入口

本次 Pi 审计按 2026-08-05 的提交 [`0df5a69`](https://github.com/earendil-works/pi/commit/0df5a69e5e119f8421f4f572d9a3c2ba4c0f5a39) 进行；后续链接仍使用 `main` 以跟随项目文档。建议从这些文件开始：

- [`packages/ai/src/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/types.ts)：Api、Provider、Model、消息和 stream options。
- [`packages/agent/src/agent-loop.ts`](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)：低层 Agent loop。
- [`packages/agent/src/agent.ts`](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent.ts)：有状态 Agent 封装。
- [`packages/coding-agent/src/core/agent-session.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/agent-session.ts)：产品级 session。
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)：session entry、branch/lane 和持久化边界。

## 和 Grok Build 怎样配合阅读

先用 Pi 读懂一条小而清楚的模型—循环—工具链，再去 Grok Build 对照 actor、权限、sandbox、leader 和长任务。对比的单位应该是“问题”：两边怎样保存历史？两边怎样处理 tool call？两边怎样把 UI 与 runtime 分开？不要只对比类名。
