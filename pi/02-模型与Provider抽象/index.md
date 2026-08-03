# 02 模型与 Provider 抽象

读完第一章的包地图，我先钻进 `pi-ai`。这一章只回答一个问题：Pi 怎么把 Anthropic、OpenAI、Gemini 这些接口差异收敛成一套统一 API，让上层代码不用关心具体厂商。

## 先看哪些文件

| 文件 | 作用 |
| --- | --- |
| [packages/ai/src/types.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/types.ts) | 核心类型：Api、Model、Context、Message、事件 |
| [packages/ai/src/models.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/models.ts) | `Provider`、`Models`、`createModels`、`createProvider` |
| [packages/ai/src/models-store.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/models-store.ts) | 模型目录缓存（动态 Provider 用） |
| [packages/ai/src/utils/event-stream.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/utils/event-stream.ts) | 事件流实现 |
| [packages/ai/src/providers/anthropic.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/providers/anthropic.ts) | 一个真实 Provider 示例 |
| [packages/ai/src/providers/faux.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/providers/faux.ts) | 测试用假 Provider，适合入门 |
| [packages/ai/README.md](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md) | 官方使用说明 |

## 为什么要统一 LLM API

Agent 要调用不同厂商的模型，而各家 API 在消息格式、流式协议、工具调用、thinking 字段上都不一样。`pi-ai` 把这些差异收敛到一组类型和一个事件流里。

[packages/ai/src/types.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/types.ts) 里先定义了一组协议标识：

```typescript
export type KnownApi =
	| "openai-completions"
	| "mistral-conversations"
	| "openai-responses"
	| "azure-openai-responses"
	| "openai-codex-responses"
	| "anthropic-messages"
	| "bedrock-converse-stream"
	| "google-generative-ai"
	| "google-vertex"
	| "pi-messages";
```

`Api` 描述的是协议/接口风格，不是厂商品牌。Anthropic 走 `anthropic-messages`，OpenAI 可以走 `openai-completions` 或 `openai-responses`。同一家公司未来换协议，只需要新增一个 `Api` 值；调用方按 `Model.api` 自动分派，不用改业务代码。

## `Model` 只是元数据

`Model` 本身不包含调用逻辑，它只是描述“哪个 Provider、什么协议、支持什么输入、上下文多大、多少钱”的元数据：

```typescript
export interface Model<TApi extends Api> {
	id: string;
	name: string;
	api: TApi;
	provider: ProviderId;
	baseUrl: string;
	reasoning: boolean;
	input: ("text" | "image")[];
	cost: ModelCost;
	contextWindow: number;
	maxTokens: number;
}
```

- `provider` 告诉系统找哪个 Provider 执行，`api` 告诉 Provider 用哪套协议。
- `contextWindow` 和 `maxTokens` 是后面上下文管理（compaction）做判断的重要依据。
- 模型列表可以静态生成、动态刷新，但调用永远通过 Provider。元数据与执行分开，是这个抽象的核心。

## `Provider` 是真正的适配器

`Provider` 负责认证、模型列表、流式调用三件事，是所有厂商差异的收口点：

```typescript
export interface Provider<TApi extends Api = Api> {
	readonly id: string;
	readonly name: string;
	readonly baseUrl?: string;
	readonly auth: ProviderAuth;
	getModels(): readonly Model<TApi>[];
	refreshModels?(context: RefreshModelsContext): Promise<void>;
	stream<T extends TApi>(
		model: Model<T>,
		context: Context,
		options?: ApiStreamOptions<T>,
	): AssistantMessageEventStream;
	streamSimple(model: Model<TApi>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}
```

- `stream` 返回 `AssistantMessageEventStream` 而不是 Promise，这是流式设计的关键，调用方可以边收边渲染。
- `auth` 是必填字段，因为每个 Provider 都要回答“我的密钥从哪来”：环境变量、凭据文件、OAuth 等。
- `getModels` 是同步读；动态 Provider 可以通过 `refreshModels` 异步更新目录。

## 一个真实 Provider 怎么组装

用 `createProvider` 把 `auth`、`models`、`api` 组装起来就得到一个 Provider。Anthropic 的实现是很合适的模板：

```typescript
export function anthropicProvider(): Provider<"anthropic-messages"> {
	return createProvider({
		id: "anthropic",
		name: "Anthropic",
		baseUrl: "https://api.anthropic.com",
		auth: {
			apiKey: anthropicApiKeyAuth(),
			oauth: lazyOAuth({ name: "Anthropic (Claude Pro/Max)", load: loadAnthropicOAuth }),
		},
		models: Object.values(ANTHROPIC_MODELS),
		api: anthropicMessagesApi(),
	});
}
```

- `anthropicApiKeyAuth` 展示了认证解析的优先级：已存凭据 → `ANTHROPIC_AUTH_TOKEN_ENV` → `ANTHROPIC_OAUTH_TOKEN_ENV` → `ANTHROPIC_API_KEY_ENV`。
- `ANTHROPIC_MODELS` 是生成的模型目录（`anthropic.models.ts`），由脚本生成以保持与官方目录一致。
- `anthropicMessagesApi()` 是协议实现，负责把统一 `Context` 翻译成 Anthropic Messages API 的请求。

## `Models` 是 Provider 的集合与统一入口

`Models` 把多个 Provider 放进一个注册表，提供“按 provider+id 查模型、自动解析认证、统一流式入口”：

```typescript
streamSimple(model: Model<Api>, context: Context, options?: ModelsSimpleStreamOptions): AssistantMessageEventStream {
	return lazyStream(model, async () => {
		const provider = this.requireProvider(model);
		const { requestModel, requestOptions } = await this.applyAuth(model, options);
		return provider.streamSimple(requestModel, context, requestOptions as SimpleStreamOptions);
	});
}
```

- `requireProvider(model)` 根据 `model.provider` 找到负责执行的 Provider。
- `applyAuth` 在真正发请求前解析认证（可能是 OAuth 刷新、环境变量读取），然后把结果放进 `requestOptions`。
- `lazyStream` 允许先拿到流对象，认证失败时再在流里报错，调用方不需要专门处理同步异常。

## 统一消息与事件协议

`Context` 与 `AssistantMessageEvent` 是 `pi-ai` 最重要的两个数据契约：

```typescript
export type AssistantMessageEvent =
	| { type: "start"; partial: AssistantMessage }
	| { type: "text_start"; contentIndex: number; partial: AssistantMessage }
	| { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
	| { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
	| { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
	| { type: "done"; reason: Extract<StopReason, "stop" | "length" | "toolUse">; message: AssistantMessage }
	| { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };
```

- 每个增量事件都携带 `partial`，也就是到目前为止的完整 `AssistantMessage`，UI 层可以无条件整体替换当前消息。
- 流必须最终以 `done` 或 `error` 结束，`error` 也携带一条 `AssistantMessage`（`stopReason: "error"` 或 `"aborted"`）。
- 这样设计后，任何 Provider 的错误都不会让调用方崩溃，而是变成一条能被 Agent 循环识别的消息。

## `EventStream` 是流式基础设施

`EventStream` 是一个自带队列、等待者和最终结果的异步迭代器，`AssistantMessageEventStream` 是它的专用子类：

```typescript
export class AssistantMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
	constructor() {
		super(
			(event) => event.type === "done" || event.type === "error",
			(event) => {
				if (event.type === "done") return event.message;
				if (event.type === "error") return event.error;
				throw new Error("Unexpected event type for final result");
			},
		);
	}
}
```

- 基类负责 push 事件、异步迭代、取最终结果三件事。
- 子类只声明两件事：什么事件算结束，结束时如何提取结果。
- 这种封装比到处回传回调函数更可组合，也是流式 API 的推荐姿势。

## 概念速查

| 概念 | 一句话解释 | 源码入口 |
| --- | --- | --- |
| `Api` | 协议风格标识 | types.ts |
| `Model` | 模型元数据，不负责执行 | types.ts |
| `Provider` | 认证 + 模型列表 + 流式执行 | models.ts |
| `Models` | Provider 注册表 + 统一入口 | models.ts |
| `Context` | 一次模型调用的输入 | types.ts |
| `AssistantMessageEventStream` | 统一流式事件协议 | event-stream.ts |

## 动手验证

1. 在 `faux.ts` 里找一个 `fauxToolCall` 示例，写一个会返回工具调用的假 Provider。
2. 用 `createModels()` 注册 Anthropic Provider，调用 `streamSimple` 打印 `text_delta`。
3. 阅读 `createProvider` 的 `dispatch` 实现，解释一个 Provider 支持多个 Api 时如何分派。

## 我还没想明白的问题

- 为什么流式事件里要携带完整 `partial` 消息而不是只携带 delta？这会提高网络带宽吗？
- `applyAuth` 失败时为什么选择在流里报错，而不是在 `streamSimple` 调用时直接抛异常？
- 动态 Provider 的模型目录刷新失败时，`refreshModels` 为什么要求保留上一次的列表？
