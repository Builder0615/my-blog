# 07 Coding Agent 与 SDK

## 本章学习目标

- 能说清 `AgentSession` 在 `Agent` 之上补了哪些能力。
- 能解释 `streamingBehavior: "steer"` 与 `"followUp"` 的区别。
- 能看懂 `pi-coding-agent` 的三种运行模式（interactive / print / rpc）。
- 知道如何通过 SDK 把 Pi 嵌入自己的应用。

## 源码地图

| 文件 | 作用 |
| --- | --- |
| [packages/coding-agent/src/core/agent-session.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/agent-session.ts) | 核心会话类：生命周期、订阅、压缩、扩展 |
| [packages/coding-agent/src/core/event-bus.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/event-bus.ts) | 轻量事件总线 |
| [packages/coding-agent/src/index.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/index.ts) | 对外导出面 |
| [packages/coding-agent/src/modes/interactive/interactive-mode.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/modes/interactive/interactive-mode.ts) | 交互 TUI 模式 |
| [packages/coding-agent/src/modes/print-mode.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/modes/print-mode.ts) | 一次性输出模式 |
| [packages/coding-agent/src/modes/rpc/rpc-mode.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/modes/rpc/rpc-mode.ts) | RPC/JSONL 模式 |
| [packages/coding-agent/docs/sdk.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md) | SDK 使用文档 |

## 正文

### 1. 内容点：`AgentSession` 是“产品级 Agent 外壳”

**结论**：`AgentSession` 在 `Agent` 之上补了持久化、扩展、模型切换、压缩、bash 执行、会话管理，是所有运行模式共享的核心类。

**源码位置**：[agent-session.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/agent-session.ts)

```typescript
export class AgentSession {
	readonly agent: Agent;
	readonly sessionManager: SessionManager;
	readonly settingsManager: SettingsManager;

	private _extensionRunner!: ExtensionRunner;
	private _resourceLoader: ResourceLoader;
	private _modelRuntime: ModelRuntime;
	private _customTools: ToolDefinition[];
	private _baseToolDefinitions: Map<string, ToolDefinition> = new Map();
	// ...

	constructor(config: AgentSessionConfig) {
		this.agent = config.agent;
		this.sessionManager = config.sessionManager;
		this.settingsManager = config.settingsManager;
		this._modelRuntime = config.modelRuntime;
		this._resourceLoader = config.resourceLoader;
		// 始终订阅 agent 事件：持久化、扩展、自动压缩、重试
		this._unsubscribeAgent = this.agent.subscribe(this._handleAgentEvent);
		this._installAgentToolHooks();
		this._installAgentNextTurnRefresh();
		this._buildRuntime({ activeToolNames: this._initialActiveToolNames, includeAllExtensionTools: true });
	}
}
```

**讲解**：

- 三个只读依赖（`agent`、`sessionManager`、`settingsManager`）说明它的定位：把执行引擎、存储、配置粘在一起。
- 构造时立刻 `agent.subscribe(_handleAgentEvent)`，所以从第一个事件开始就会做持久化。
- `_buildRuntime` 把扩展工具、内置工具、模型运行时装配成 Agent 的最终配置。

### 2. 内容点：`prompt()` 的流式行为由调用方声明

**结论**：当 Agent 正在运行时，`prompt()` 必须显式声明 `streamingBehavior`：`"steer"` 表示立即插入指令，`"followUp"` 表示排队等干完再执行。

**源码位置**：[agent-session.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/agent-session.ts)

```typescript
export interface PromptOptions {
	/** 是否展开文件型 prompt template（默认 true） */
	expandPromptTemplates?: boolean;
	/** 图片附件 */
	images?: ImageContent[];
	/** 流式时如何排队："steer"（打断）或 "followUp"（等待）。流式时必填。 */
	streamingBehavior?: "steer" | "followUp";
	/** 输入来源，供扩展 input 事件使用 */
	source?: InputSource;
	/** RPC 模式用来观察 preflight 是否被接受 */
	preflightResult?: (success: boolean) => void;
}
```

**讲解**：

- `steer` 对应低层循环里的 `getSteeringMessages`：消息在下一轮模型调用前注入。
- `followUp` 对应 `getFollowUpMessages`：等 Agent 自然结束后再处理。
- 学习价值：产品层把“插话/排队”语义变成显式参数，比让用户猜行为更可靠。

### 3. 内容点：事件订阅既是 UI 管道，也是持久化管道

**结论**：`subscribe()` 返回取消函数；内部通过同一个订阅机制完成会话落盘，保证 UI 看到的和存下来的是一致的。

**源码位置**：[agent-session.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/agent-session.ts)

```typescript
subscribe(listener: AgentSessionEventListener): () => void {
	// 加入用户监听器，返回 unsubscribe
}
```

**讲解**：

- `_handleAgentEvent` 是内部监听器：负责持久化、扩展事件转发、自动压缩检查、重试状态更新。
- 用户监听器在内部监听器之后收到事件，因此能看到已经持久化的状态。
- 这种“内部消费者优先”的顺序保证了持久化先于 UI 更新，崩溃时 UI 状态不会超前于磁盘。

### 4. 内容点：三种运行模式共享同一颗核心

**结论**：`interactive-mode`（TUI）、`print-mode`（一次性）、`rpc-mode`（JSONL over stdio）都只做“输入输出适配”，业务逻辑全在 `AgentSession`。

**源码位置**：[packages/coding-agent/src/modes](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/modes)

```text
modes/
├── interactive/interactive-mode.ts   # 全屏 TUI：编辑器、进度、快捷键
├── print-mode.ts                     # 单次提问，输出到 stdout
└── rpc/rpc-mode.ts                   # stdin/stdout JSONL 协议
```

**讲解**：

- 这是“核心逻辑与 I/O 分离”的教科书级示范：未来新增 IDE 集成时，只需要再写一个 mode。
- RPC 模式是 IDE/自动化集成的关键入口，配合 `pi-protocol` 或 JSONL 使用。

### 5. 内容点：SDK 让第三方应用直接创建会话

**结论**：SDK 暴露 `createAgentSession`，应用只需要提供模型运行时和 session manager 就能嵌入 Agent。

**源码位置**：[sdk.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md)

```typescript
import { createAgentSession, ModelRuntime, SessionManager } from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create();
const { session } = await createAgentSession({
	sessionManager: SessionManager.inMemory(),
	modelRuntime,
});

session.subscribe((event) => {
	if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
		process.stdout.write(event.assistantMessageEvent.delta);
	}
});

await session.prompt("What files are in the current directory?");
```

**讲解**：

- `createAgentSession` 隐藏了资源加载、系统提示、工具注册等细节。
- `SessionManager.inMemory()` 让测试和简单场景不需要文件系统。
- 学习价值：一个好的 Agent 框架，最终要能像这样“几行代码启动”。

### 6. 内容点：扩展系统通过事件和 API 双向注入

**结论**：`pi-coding-agent` 的扩展可以注册工具、命令、快捷键、UI 组件，并监听生命周期事件。

**源码位置**：[index.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/index.ts)

```typescript
export {
	defineTool,
	discoverAndLoadExtensions,
	ExtensionRunner,
	type Extension,
	type ExtensionFactory,
	type ExtensionAPI,
	type ToolDefinition,
	// ...
} from "./core/extensions/index.ts";
```

**讲解**：

- 扩展核心概念是 `ExtensionFactory` + `ExtensionAPI`：工厂拿到运行环境，返回扩展对象。
- `defineTool` 把普通函数包装成可被模型调用的工具定义。
- 目录里还有 `examples/extensions`（subagent、plan-mode 等），适合照着做。

## 小结

| 层 | 职责 | 关键类/文件 |
| --- | --- | --- |
| Agent | 循环与工具 | `Agent` |
| Session | 生命周期、持久化、扩展 | `AgentSession` |
| Mode | I/O 适配 | interactive / print / rpc |
| SDK | 对外嵌入入口 | `createAgentSession` |
| Extension | 能力扩展 | `defineTool` / `ExtensionRunner` |

## 练习与思考

1. 读 `_handleAgentEvent`，列出它会响应哪些事件，以及每个事件触发什么副作用。
2. 在 SDK quick start 上增加一个自定义工具，验证工具定义如何到达模型。
3. 对比 `print-mode` 与 `rpc-mode` 的输入输出形态，说明 RPC 为什么更适合 IDE。

## 延伸问题

- 为什么 `subscribe` 的取消函数要保留内部监听器重连逻辑（`resubscribe`）？
- `streamingBehavior` 不传时为什么直接报错而不是默认排队？
- 扩展系统为什么把事件分成“观察型”和“可修改型”（hook），而不是统一回调？
