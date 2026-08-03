# 03 Agent 核心循环

## 本章学习目标

- 能画出 Agent 循环的状态转移图，说清内层循环和外层循环的区别。
- 能解释 `AgentMessage` 与 LLM `Message` 为什么需要转换层。
- 能说清 `agentLoop`、`runAgentLoop`、`Agent` 三者的分工。
- 能读懂 `streamAssistantResponse` 的事件处理逻辑。

## 源码地图

| 文件 | 作用 |
| --- | --- |
| [packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts) | 低层循环：事件发射、模型调用、工具执行 |
| [packages/agent/src/agent.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent.ts) | 有状态的 `Agent` 封装：订阅、队列、运行管理 |
| [packages/agent/src/types.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/types.ts) | `AgentMessage`、`AgentTool`、`AgentLoopConfig` |
| [packages/agent/src/index.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/index.ts) | 包导出，看边界 |

## 正文

### 1. 内容点：一次对话在事件层面长什么样

**结论**：Pi 把一次 `prompt()` 展开成一串标准事件，UI、持久化、扩展都订阅这些事件；这是“可观察 Agent”的核心。

**源码位置**：[packages/agent/README.md](https://github.com/earendil-works/pi/blob/main/packages/agent/README.md)

```text
prompt("Hello")
├─ agent_start
├─ turn_start
├─ message_start / message_end   // 用户消息
├─ message_start / message_update / message_end  // 助手消息（流式）
├─ turn_end
└─ agent_end
```

带工具调用时，中间会出现：

```text
├─ tool_execution_start   { toolCallId, toolName, args }
├─ tool_execution_update  { partialResult }
├─ tool_execution_end     { toolCallId, result }
├─ message_start/end      { toolResultMessage }
└─ turn_end
```

**讲解**：

- `agent_start/agent_end` 标记一次完整运行的生命周期。
- `turn_start/turn_end` 标记“一次模型调用 + 它引发的工具执行”。
- `message_update` 是 UI 渲染的燃料，携带 `assistantMessageEvent` 增量。

### 2. 内容点：`agentLoop` 是低层循环的入口

**结论**：`agentLoop` 接收 prompt、上下文和配置，返回一个 `EventStream<AgentEvent, AgentMessage[]>`，让调用方既能迭代事件，也能拿到最终消息列表。

**源码位置**：[packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

```typescript
export function agentLoop(
	prompts: AgentMessage[],
	context: AgentContext,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): EventStream<AgentEvent, AgentMessage[]> {
	const stream = createAgentStream();

	void runAgentLoop(
		prompts,
		context,
		config,
		async (event) => {
			stream.push(event);
		},
		signal,
		streamFn,
	).then((messages) => {
		stream.end(messages);
	});

	return stream;
}
```

**讲解**：

- `createAgentStream()` 声明“`agent_end` 事件出现即结束”，并从该事件里提取 `messages`。
- 真正的执行在 `runAgentLoop` 里；`agentLoop` 只是把“异步执行”接到“事件流”上。
- 学习价值：把“执行”和“消费”解耦，消费方不需要 `await` 整个运行，就可以边收边渲染。

### 3. 内容点：内外两层循环各管什么

**结论**：内层循环处理“工具调用与 steering 消息”，外层循环处理“follow-up 消息”，两者合起来就是完整的多轮工具型 Agent。

**源码位置**：[packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

```typescript
// 外层循环：follow-up 消息
while (true) {
	let hasMoreToolCalls = true;

	// 内层循环：工具调用 + steering
	while (hasMoreToolCalls || pendingMessages.length > 0) {
		if (!firstTurn) {
			await emit({ type: "turn_start" });
		} else {
			firstTurn = false;
		}

		if (pendingMessages.length > 0) {
			for (const message of pendingMessages) {
				await emit({ type: "message_start", message });
				await emit({ type: "message_end", message });
				currentContext.messages.push(message);
				newMessages.push(message);
			}
			pendingMessages = [];
		}

		const message = await streamAssistantResponse(...);
		const toolCalls = message.content.filter((c) => c.type === "toolCall");

		const toolResults: ToolResultMessage[] = [];
		hasMoreToolCalls = false;
		if (toolCalls.length > 0) {
			const executedToolBatch =
				message.stopReason === "length"
					? await failToolCallsFromTruncatedMessage(toolCalls, emit)
					: await executeToolCalls(currentContext, message, config, signal, emit);
			toolResults.push(...executedToolBatch.messages);
			hasMoreToolCalls = !executedToolBatch.terminate;
		}

		await emit({ type: "turn_end", message, toolResults });

		// 检查是否该停、是否要更新下一轮配置、是否有 steering
		if (await config.shouldStopAfterTurn?.(...)) {
			await emit({ type: "agent_end", messages: newMessages });
			return;
		}
		pendingMessages = (await config.getSteeringMessages?.()) || [];
	}

	const followUpMessages = (await config.getFollowUpMessages?.()) || [];
	if (followUpMessages.length > 0) {
		pendingMessages = followUpMessages;
		continue;
	}
	break;
}

await emit({ type: "agent_end", messages: newMessages });
```

**讲解**：

- `steering` 是“Agent 正在工作时插入的指令”（例如用户说“改成这样”），内层循环每次模型回复后都会检查。
- `followUp` 是“等 Agent 干完活再追加的任务”，所以放在外层循环，只有当内层完全停止时才检查。
- `shouldStopAfterTurn` 可以让调用方在“下一轮调用模型之前”优雅收尾，例如上下文快满了。
- `prepareNextTurn` 可以替换下一轮的 `model` / `thinkingLevel` / `context`，实现运行中换模型。

可以用状态图表示：

```mermaid
stateDiagram-v2
    [*] --> 等待消息
    等待消息 --> 调用模型: 收到 prompt/steering/follow-up
    调用模型 --> 有工具调用: stopReason=toolUse
    调用模型 --> 完成: stopReason=stop
    有工具调用 --> 执行工具
    执行工具 --> 调用模型: 注入 toolResult
    执行工具 --> 完成: terminate=true
    完成 --> 有follow-up: 队列非空
    完成 --> [*]: 无follow-up
    有follow-up --> 调用模型
```

### 4. 内容点：`streamAssistantResponse` 是“转换 + 调用 + 归并”的三段式

**结论**：每次模型调用前，Agent 先把内部 `AgentMessage[]` 转成 LLM 认识的 `Message[]`，调用后再把流式事件归并成一条完整 `AssistantMessage` 放进上下文。

**源码位置**：[packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

```typescript
async function streamAssistantResponse(...): Promise<AssistantMessage> {
	let messages = context.messages;
	if (config.transformContext) {
		messages = await config.transformContext(messages, signal);
	}

	const llmMessages = await config.convertToLlm(messages);

	const llmContext: Context = {
		systemPrompt: context.systemPrompt,
		messages: llmMessages,
		tools: context.tools,
	};

	const resolvedApiKey =
		(config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;

	const response = await streamFunction(config.model, llmContext, {
		...config,
		apiKey: resolvedApiKey,
		signal,
	});
	// ... 遍历事件，维护 partialMessage
}
```

**讲解**：

- 顺序是 `AgentMessage[] → transformContext → convertToLlm → Context → streamFunction`。
- `transformContext` 处理“内部消息如何剪裁/增强”，`convertToLlm` 处理“如何翻译给模型”。
- `getApiKey` 允许动态刷新短时效的 OAuth token，是长任务里很实用的设计。

### 5. 内容点：事件归并时如何保证上下文一致

**结论**：流式期间，循环先把 `partial` 消息放入 `context.messages`，收到 `done` 后替换为最终消息，保证订阅者与上下文看到的始终是最新状态。

**源码位置**：[packages/agent/src/agent-loop.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

```typescript
case "start":
	partialMessage = event.partial;
	context.messages.push(partialMessage);
	addedPartial = true;
	await emit({ type: "message_start", message: { ...partialMessage } });
	break;

case "text_delta":
case "toolcall_delta":
	// ...
	if (partialMessage) {
		partialMessage = event.partial;
		context.messages[context.messages.length - 1] = partialMessage;
		await emit({ type: "message_update", assistantMessageEvent: event, message: { ...partialMessage } });
	}
	break;
```

**讲解**：

- `start` 时把 partial 消息“压入”上下文，后续每个 delta 都“替换”最后一个元素。
- `done/error` 时用 `response.result()` 拿到最终消息并替换。
- 这就是为什么事件里要携带完整 `partial`：循环侧只需要做“赋值”而不是“拼接”。

### 6. 内容点：`Agent` 类是低层循环的有状态封装

**结论**：`Agent` 拥有消息历史、订阅者集合、steering/follow-up 队列和活动运行管理，适合“长期存在、可多次 prompt”的场景。

**源码位置**：[packages/agent/src/agent.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent.ts)

```typescript
export class Agent {
	private _state: MutableAgentState;
	private readonly listeners = new Set<...>();
	private readonly steeringQueue: PendingMessageQueue;
	private readonly followUpQueue: PendingMessageQueue;
	// ...

	constructor(options: AgentOptions) {
		this._state = createMutableAgentState(runtimeOptions.initialState);
		this.convertToLlm = runtimeOptions.convertToLlm ?? defaultConvertToLlm;
		this.steeringQueue = new PendingMessageQueue(runtimeOptions.steeringMode ?? "one-at-a-time");
		this.followUpQueue = new PendingMessageQueue(runtimeOptions.followUpMode ?? "one-at-a-time");
		this.toolExecution = runtimeOptions.toolExecution ?? "parallel";
	}
}
```

**讲解**：

- `PendingMessageQueue` 的 `drain()` 根据 `"one-at-a-time"` / `"all"` 决定一次注入几条消息。
- `Agent` 把低层循环的“函数参数”变成了“对象状态”，并负责把队列里的消息喂给循环的 `getSteeringMessages` / `getFollowUpMessages`。
- 学习重点：同样是循环，`agentLoop` 是函数式的、一次性的；`Agent` 是有状态的、可复用的。上层 `AgentSession` 再在这个基础上加持久化。

### 7. 内容点：`convertToLlm` 的默认实现为什么是“过滤”

**结论**：`AgentMessage` 比 LLM 消息更宽泛（可以有自定义消息类型），默认转换只保留 LLM 能识别的三种角色，其余类型被过滤。

**源码位置**：[packages/agent/src/agent.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent.ts)

```typescript
function defaultConvertToLlm(messages: AgentMessage[]): Message[] {
	return messages.filter(
		(message) => message.role === "user" || message.role === "assistant" || message.role === "toolResult",
	);
}
```

**讲解**：

- 自定义消息类型（例如“状态通知”“系统事件”）可以存在会话里，但不会发给模型。
- 调用方可以传入自己的 `convertToLlm`，把自定义消息翻译成用户消息等，实现“UI 消息也能影响模型”的效果。

## 小结

| 概念 | 作用 | 源码 |
| --- | --- | --- |
| `agentLoop` | 函数式低层循环入口 | agent-loop.ts |
| `runLoop` | 内层（工具/steering）+ 外层（follow-up） | agent-loop.ts |
| `streamAssistantResponse` | 转换上下文 → 调模型 → 归并事件 | agent-loop.ts |
| `Agent` | 有状态封装、订阅、队列 | agent.ts |
| `AgentEvent` | 生命周期/流式事件契约 | types.ts |

## 练习与思考

1. 在 `agent-loop.ts` 里用 `rg "await emit"` 列出所有事件，画出一张事件时序图。
2. 给 `Agent` 配置 `beforeToolCall` 并 `block` 一个工具，观察事件序列里是否出现错误 toolResult。
3. 自己实现一个 `convertToLlm`，让一种自定义消息类型变成 user 消息，再对比默认过滤行为。

## 延伸问题

- `stopReason === "length"` 时为什么要把所有工具调用全部标记失败，而不是执行一部分？
- `prepareNextTurn` 和 `shouldStopAfterTurn` 的执行顺序为什么是“先更新快照、再判断停止”？
- 如果 steering 消息在模型调用中途到达，为什么 Pi 不打断当前流式响应，而是等下一轮注入？
