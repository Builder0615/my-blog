# 09 协议、存储与评测

这一章把 Pi 里三个比较“外围”的包放在一起看：`pi-protocol` 管跨进程通信，`pi-storage` 管会话持久化，`pi-evals` 管行为回归。它们不像核心循环那么显眼，但都是把一个 Agent 变成产品的必要条件。

## 先看哪些文件

| 文件 | 作用 |
| --- | --- |
| [packages/protocol/README.md](https://github.com/earendil-works/pi/blob/main/packages/protocol/README.md) | 协议设计说明 |
| [packages/protocol/src/framing.ts](https://github.com/earendil-works/pi/blob/main/packages/protocol/src/framing.ts) | 4 字节长度前缀 + 帧解码 |
| [packages/protocol/src/schemas.ts](https://github.com/earendil-works/pi/blob/main/packages/protocol/src/schemas.ts) | 严格消息 schema |
| [packages/storage/sqlite-node/src/sqlite/migrations/001_initial.sql](https://github.com/earendil-works/pi/blob/main/packages/storage/sqlite-node/src/sqlite/migrations/001_initial.sql) | 存储表结构 |
| [packages/evals/README.md](https://github.com/earendil-works/pi/blob/main/packages/evals/README.md) | 评测框架说明 |
| [packages/evals/src/pi-harness.ts](https://github.com/earendil-works/pi/blob/main/packages/evals/src/pi-harness.ts) | Pi 评测 harness |
| [packages/evals/src/smoke.eval.ts](https://github.com/earendil-works/pi/blob/main/packages/evals/src/smoke.eval.ts) | 最小评测示例 |

## 协议 = 帧 + CBOR 载荷

每条协议消息是 `[4 字节大端长度][CBOR 载荷]`。帧解决消息边界，CBOR 解决紧凑编码与严格校验：

```typescript
export function encodeFrame(payload: Uint8Array): Uint8Array {
	if (payload.byteLength > MAX_UINT32) throw new RangeError("Frame payload exceeds the unsigned 32-bit length limit");
	const frame = new Uint8Array(FRAME_HEADER_LENGTH + payload.byteLength);
	const length = payload.byteLength;
	frame[0] = length >>> 24;
	frame[1] = length >>> 16;
	frame[2] = length >>> 8;
	frame[3] = length;
	frame.set(payload, FRAME_HEADER_LENGTH);
	return frame;
}
```

- 大端 4 字节长度让接收端可以增量解析任意分片或合并的字节流。
- `FrameDecoder` 处理“一个 chunk 可能包含多个帧、或只包含半个帧”的流式情况。

## schema 刻意做成“严格模式”

协议消息用 TypeBox 定义，并默认 `additionalProperties: false`，未知字段直接拒绝，保证跨语言实现不会产生歧义：

```typescript
const StrictObject = <const T extends Parameters<typeof Type.Object>[0]>(properties: T) =>
	Type.Object(properties, { additionalProperties: false });

export const ModelRefSchema = StrictObject({
	provider: IdSchema,
	id: IdSchema,
});
```

- 拒绝未知字段意味着新旧版本之间需要显式版本升级，而不是静默吞掉。
- 跨进程、跨语言协议里，严格 schema 是减少 bug 最便宜的手段。

## SQLite 表设计服务于“分支读取”

会话原始数据进 `session_entries`，分支关系进 `branch_entries`，物化视图加速常用查询：

```sql
CREATE TABLE IF NOT EXISTS branch_entries (
	session_id TEXT NOT NULL,
	branch_id TEXT NOT NULL,
	entry_id TEXT NOT NULL,
	entry_seq INTEGER NOT NULL,
	PRIMARY KEY (session_id, branch_id, entry_id)
);

CREATE INDEX IF NOT EXISTS idx_branch_entries_session_branch_seq
	ON branch_entries(session_id, branch_id, entry_seq);
```

- `branch_id + entry_seq` 索引让按分支顺序读变成顺序扫描。
- `session_materialized` 存整条会话的快照载荷，适合频繁读取的场景。
- 和 JSONL 后端比，SQLite 的查询能力（FTS、物化视图）更强，代价是存储结构更复杂。

## 评测用真实 Agent 会话而不是 mock

`pi-harness.ts` 创建真实的 `AgentSession`，把 prompt 跑成真实消息序列，再转成 `vitest-evals` 的 transcript 做断言：

```typescript
async function promptAgent(session: AgentSession, input: string, signal: AbortSignal | undefined): Promise<string> {
	signal?.throwIfAborted();
	const previousMessageCount = session.messages.length;
	await session.prompt(input);
	const assistant = session.messages
		.slice(previousMessageCount)
		.reverse()
		.find((message) => message.role === "assistant");
	if (!assistant) throw new Error("Agent run completed without an assistant message.");
	if (assistant.stopReason !== "stop") {
		throw new Error(assistant.errorMessage ?? `Agent run ended with unexpected stop reason: ${assistant.stopReason}.`);
	}
	const output = session.getLastAssistantText();
	if (!output) throw new Error("Agent run produced no assistant text.");
	return output;
}
```

- 评测直接驱动真实会话，才能测出 prompt、工具、skill、model 之间的真实交互。
- `toTranscriptEvents` 把会话消息转成 tool_call / tool_result 等结构化事件，方便断言模型行为。

## 最小评测长这样

用 `describeEval` + harness，一个断言就能验证“端到端能跑通”：

```typescript
import { expect } from "vitest";
import { describeEval } from "vitest-evals";
import { createPiCodingAgentHarness } from "./pi-harness.ts";

const piCodingAgentHarness = createPiCodingAgentHarness({ noTools: "all" });

describeEval("Pi Coding Agent smoke", { harness: piCodingAgentHarness }, (it) => {
	it("runs a basic prompt end to end", async ({ run }) => {
		const result = await run("What's the capital of France? Respond with only the city name.");
		expect(result.output.trim()).toBe("Paris");
		expect(result.errors).toEqual([]);
		expect(result.usage.totalTokens).toBeGreaterThan(0);
	});
});
```

运行方式见 [evals/README.md](https://github.com/earendil-works/pi/blob/main/packages/evals/README.md)：

```bash
npm run eval -- --provider openai --model gpt-5.6-sol
```

Agent 项目的回归测试离不开真实模型评测，这套结构可以直接借到自己的项目里。

## 概念速查

| 模块 | 解决什么问题 | 关键设计 |
| --- | --- | --- |
| `pi-protocol` | 跨进程通信 | 帧 + CBOR + 严格 schema |
| `pi-storage` | 会话持久化 | entry 树 + 分支索引 + 物化视图 |
| `pi-evals` | 行为回归 | 真实会话 + transcript + 断言 |

## 动手验证

1. 用 `encodeFrame` 手工构造一帧，再用 `FrameDecoder` 分两次喂入，观察解码结果。
2. 在 SQLite 表结构里找“分支读取”需要的全部索引，说明查询路径。
3. 给 `smoke.eval.ts` 增加一个“必须调用指定工具”的断言。

## 我还没想明白的问题

- 为什么协议版本号放在消息 schema 里而不是只靠包版本？
- `branch_entries` 和 `session_entries.entry_seq` 各承担什么职责？可以只留一个吗？
- 评测里 `noTools: "all"` 关闭全部工具后，为什么还能验证“端到端”？
