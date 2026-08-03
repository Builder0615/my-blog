# 04 工具调用与参数校验

## 本章学习目标

- 能说清 `Tool`、`AgentTool`、`ToolCall`、`ToolResultMessage` 的关系。
- 能解释 TypeBox schema 为什么用于工具参数定义。
- 能读懂 `prepareToolCall` 中的“找不到工具 / 参数非法 / 被 hook 阻止”三条失败路径。
- 能对比 `sequential` 与 `parallel` 两种执行模式的行为差异。

## 源码地图

| 文件 | 作用 |
| --- | --- |
| [packages/ai/src/types.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/types.ts) | `Tool`、`ToolCall`、`ToolResultMessage` |
| [packages/ai/src/utils/validation.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/utils/validation.ts) | 参数校验与类型纠正 |
| [packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts) | 工具执行：prepare / execute / finalize |
| [packages/agent/src/types.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/types.ts) | `AgentTool`、hook 类型 |
| [packages/agent/src/harness/tools](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/tools) | 内置 read/write/edit/bash 工具 |

## 正文

### 1. 内容点：工具在“模型侧”只是一个 schema 描述

**结论**：对 LLM 来说，工具就是“名字 + 描述 + 参数 JSON Schema”；真正的执行逻辑在 Agent 侧。

**源码位置**：[packages/ai/src/types.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/types.ts)

```typescript
export interface Tool<TParameters extends TSchema = TSchema> {
	name: string;
	description: string;
	parameters: TParameters;
	constrainedSampling?: false | ConstrainedSamplingConfig;
}
```

**讲解**：

- `parameters` 是 TypeBox schema（运行时校验 + JSON Schema 转换）。
- `constrainedSampling` 是可选的高级能力，用于要求模型按指定格式生成参数（如 strict JSON schema 或 grammar）。
- Agent 把这些 `Tool[]` 放进 `Context.tools`，模型才能在回复里发出 `toolCall`。

### 2. 内容点：模型返回的 `ToolCall` 是回复内容块

**结论**：`AssistantMessage.content` 是内容块数组，其中 `type: "toolCall"` 的块就是一个工具调用请求。

**源码位置**：[packages/ai/src/types.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/types.ts)

```typescript
export interface ToolCall {
	type: "toolCall";
	id: string;
	name: string;
	arguments: Record<string, any>;
	thoughtSignature?: string;
}
```

**讲解**：

- 一条助手消息可以包含多个 `toolCall`，这就催生了“并行执行多个工具”的需求。
- `arguments` 是“模型可能没按 schema 生成”的原始参数，必须经过校验和纠正后才安全。

### 3. 内容点：`AgentTool` 是 Agent 侧的“可执行工具”

**结论**：`AgentTool` 在 `Tool` 基础上增加了 `execute` 函数、参数预处理、流式更新回调和执行模式。

**源码位置**：[packages/agent/src/types.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/types.ts)

```typescript
export interface AgentTool<TParameters extends TSchema = TSchema, TResult = unknown, TContext = undefined>
	extends Tool<TParameters> {
	execute(
		toolCallId: string,
		args: Static<TParameters>,
		signal: AbortSignal | undefined,
		update: (partialResult: Partial<TResult>) => void,
	): Promise<TResult>;
	prepareArguments?: (args: Record<string, any>) => Record<string, any>;
	executionMode?: ToolExecutionMode;
}
```

**讲解**：

- `execute` 接收校验后的参数（`Static<TParameters>`），返回工具结果；通过 `update` 回传流式进度。
- `prepareArguments` 允许在“校验之前”修正模型给的参数（例如把字符串路径转成对象）。
- `executionMode: "sequential"` 会强制整批工具串行执行。

### 4. 内容点：校验前先“纠正”，能救回很多模型手滑

**结论**：`validateToolArguments` 不止校验，还会对常见类型做尽力纠正，例如把字符串 `"3"` 转成数字 3。

**源码位置**：[packages/ai/src/utils/validation.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/utils/validation.ts)

```typescript
function coercePrimitiveByType(value: unknown, type: string): unknown {
	switch (type) {
		case "number": {
			if (value === null) return 0;
			if (typeof value === "string" && value.trim() !== "") {
				const parsed = Number(value);
				if (Number.isFinite(parsed)) return parsed;
			}
			if (typeof value === "boolean") return value ? 1 : 0;
			return value;
		}
		// ...
	}
}
```

**讲解**：

- 模型的 JSON 工具参数经常出现“类型漂移”，例如数字被序列化成字符串。
- 先纠正再校验，比直接拒绝更宽容，能减少“模型改口重试”的成本。
- 但纠正不是无条件的，无法转换的值仍会原样返回，最终由校验决定成败。

### 5. 内容点：`prepareToolCall` 是工具执行的“安检门”

**结论**：一个工具调用在执行前要经过四道检查：工具是否存在、参数是否合法、hook 是否放行、信号是否已中止。

**源码位置**：[packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

```typescript
async function prepareToolCall(currentContext, assistantMessage, toolCall, config, signal) {
	const tool = currentContext.tools?.find((t) => t.name === toolCall.name);
	if (!tool) {
		return { kind: "immediate", result: createErrorToolResult(`Tool ${toolCall.name} not found`), isError: true };
	}

	try {
		const preparedToolCall = prepareToolCallArguments(tool, toolCall);
		const validatedArgs = validateToolArguments(tool, preparedToolCall);

		if (config.beforeToolCall) {
			const beforeResult = await config.beforeToolCall({ ... }, signal);
			if (beforeResult?.block) {
				return { kind: "immediate", result: createErrorToolResult(beforeResult.reason || "Tool execution was blocked"), isError: true };
			}
		}
		// ...
		return { kind: "prepared", toolCall, tool, args: validatedArgs };
	} catch (error) {
		return { kind: "immediate", result: createErrorToolResult(...), isError: true };
	}
}
```

**讲解**：

- “失败”不是抛异常，而是返回一条 `isError: true` 的 `ToolResultMessage`，模型会看到错误并自行决定下一步。
- `beforeToolCall` 是安全/策略钩子，例如“禁止 bash 工具”就在这一层实现。
- 四道检查全部通过，才返回 `kind: "prepared"` 进入真正的执行。

### 6. 内容点：执行、后处理、事件发射是分开的三步

**结论**：Pi 把工具调用拆成 `executePreparedToolCall`（执行）、`finalizeExecutedToolCall`（后处理）、`emitToolExecutionEnd`（发布事件），每一步都能独立测试。

**源码位置**：[packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

```typescript
async function executePreparedToolCall(prepared, signal, emit) {
	const updateEvents: Promise<void>[] = [];
	let acceptingUpdates = true;
	try {
		const result = await prepared.tool.execute(
			prepared.toolCall.id,
			prepared.args,
			signal,
			(partialResult) => {
				if (!acceptingUpdates) return;
				updateEvents.push(Promise.resolve(emit({ type: "tool_execution_update", ... })));
			},
		);
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		return { result, isError: false };
	} catch (error) {
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		return { result: createErrorToolResult(...), isError: true };
	}
}
```

**讲解**：

- `acceptingUpdates` 防止工具在返回结果之后继续乱发更新事件。
- 工具抛异常会被转成 `isError: true` 的结果，而不是让整个 Agent 崩溃。
- `afterToolCall` 可以在执行后改写结果（替换内容、标记终止等），例如清理敏感输出。

### 7. 内容点：`sequential` 与 `parallel` 的差异

**结论**：串行模式逐个执行；并行模式先逐个“preflight”，再并发执行，最后按助手消息里的原始顺序生成 toolResult 消息。

**源码位置**：[packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

```typescript
async function executeToolCalls(currentContext, assistantMessage, config, signal, emit) {
	const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");
	const hasSequentialToolCall = toolCalls.some(
		(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
	);
	if (config.toolExecution === "sequential" || hasSequentialToolCall) {
		return executeToolCallsSequential(...);
	}
	return executeToolCallsParallel(...);
}
```

**讲解**：

- 并行模式下，`tool_execution_start` 仍按顺序发射（preflight），`tool_execution_end` 按完成顺序发射。
- 但写进上下文的 `ToolResultMessage` 会回到“助手消息里的原始顺序”，避免模型看到顺序错乱。
- 任何一个工具声明 `executionMode: "sequential"`，整批都会退化为串行，保证依赖顺序。

### 8. 内容点：输出被截断时，工具调用全部作废

**结论**：当模型因 `stopReason === "length"` 被截断时，流式工具参数可能不完整；Pi 选择一个都不执行，全部返回错误让模型重发。

**源码位置**：[packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

```typescript
const executedToolBatch =
	message.stopReason === "length"
		? await failToolCallsFromTruncatedMessage(toolCalls, emit)
		: await executeToolCalls(...);
```

**讲解**：

- 截断时虽然参数能“拼出一个合法 JSON”，但语义上可能少了一半，执行有风险。
- 错误信息明确提示模型“参数可能被截断，请重新完整发出”，让模型自愈。

### 9. 内容点：内置工具通过 `ExecutionEnv` 与外部环境解耦

**结论**：`pi-agent-core` 提供 read/write/edit/bash 工具，但它们不直接碰 `node:fs` / 子进程，而是通过 `ExecutionEnv` 抽象执行。

**源码位置**：[packages/agent/src/harness/tools/index.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/tools/index.ts)

```typescript
export {
	createReadTool,
	createWriteTool,
	createEditTool,
	createBashTool,
} from "./read.ts";
```

**讲解**：

- 具体实现见 `read.ts` / `write.ts` / `edit.ts` / `bash.ts`，它们都接收 `ExecutionToolContext` 里的 `env: ExecutionEnv`。
- 这层抽象让同一套工具既能在本地 Node 运行，也能被沙箱、测试替身替换，是“工具可测试化”的范例。

## 小结

| 阶段 | 函数 | 关键行为 |
| --- | --- | --- |
| 描述 | `Tool` | schema + 描述 |
| 解析 | `prepareToolCallArguments` | 预处理参数 |
| 校验 | `validateToolArguments` | 纠正类型 + 校验 |
| 策略 | `beforeToolCall` | 可阻止执行 |
| 执行 | `executePreparedToolCall` | 真正调用，异常转结果 |
| 后处理 | `afterToolCall` | 改写结果/终止 |
| 落库 | `createToolResultMessage` | 生成 toolResult 消息 |

## 练习与思考

1. 用 `Type.Object` 定义一个工具，故意让模型返回类型错误的参数，观察校验纠正。
2. 给 `Agent` 配 `beforeToolCall` 阻止 `bash`，确认模型收到的错误信息。
3. 读 `executeToolCallsParallel`，说明 `FinalizedToolCallEntry` 为什么是“结果或函数”的联合类型。

## 延伸问题

- 为什么“并行执行”也要先顺序 preflight？先并发 preflight 会引入什么问题？
- `terminate: true` 需要“整批都为 true”才停止，这个规则避免了什么风险？
- 工具返回 `addedToolNames` 的用途是什么？它在哪个 Provider 场景下真正生效？
