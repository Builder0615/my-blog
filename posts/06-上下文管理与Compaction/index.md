# 06 上下文管理与 Compaction

## 本章学习目标

- 能解释 Agent 上下文为什么必然增长，以及增长后有什么后果。
- 能说清 `shouldCompact`、`prepareCompaction`、`compact` 的分工。
- 能理解“保留 recent tail + 总结历史”为什么比“全量删除旧消息”更好。
- 能说明 `compaction` entry 在会话树里如何改变后续上下文构建。

## 源码地图

| 文件 | 作用 |
| --- | --- |
| [packages/agent/src/harness/compaction/compaction.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/compaction/compaction.ts) | 阈值判断、切点、总结、落盘准备 |
| [packages/agent/src/harness/session/session.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/session/session.ts) | 上下文构建与 compaction 变换 |
| [packages/agent/src/harness/agent-harness.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/agent-harness.ts) | `transformContext` 与 auto compaction 触发 |
| [packages/coding-agent/docs/compaction.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/compaction.md) | 用户视角的压缩说明 |

## 正文

### 1. 内容点：Token 估算是一切决策的基础

**结论**：Pi 不依赖 Provider 实时报告，而是先用保守的字符启发式估算每条消息的 token 数，用于判断“上下文快满了”。

**源码位置**：[compaction.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/compaction/compaction.ts)

```typescript
export function estimateTokens(message: AgentMessage): number {
	let chars = 0;
	switch (message.role) {
		case "user":
			chars = estimateTextAndImageContentChars(message.content);
			return Math.ceil(chars / 4);
		case "assistant":
			for (const block of assistant.content) {
				if (block.type === "text") chars += block.text.length;
				else if (block.type === "thinking") chars += block.thinking.length;
				else if (block.type === "toolCall") chars += block.name.length + safeJsonStringify(block.arguments).length;
			}
			return Math.ceil(chars / 4);
		// ...
	}
	return 0;
}
```

**讲解**：

- 图片按固定字符数估算，避免把整张 base64 塞进估算。
- “保守”意味着宁可早压缩，不可晚到溢出。

### 2. 内容点：压缩阈值是一个“水位线”

**结论**：`shouldCompact` 判断当前 token 是否超过 `contextWindow - reserveTokens`；`reserveTokens` 是给总结 prompt 和输出留出的余量。

**源码位置**：[compaction.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/compaction/compaction.ts)

```typescript
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
	enabled: true,
	reserveTokens: 16384,
	keepRecentTokens: 20000,
};

export function shouldCompact(contextTokens: number, contextWindow: number, settings: CompactionSettings): boolean {
	if (!settings.enabled) return false;
	return contextTokens > contextWindow - settings.reserveTokens;
}
```

**讲解**：

- `keepRecentTokens` 表示压缩后保留的“近期上下文”预算。
- 阈值判断只管“要不要压缩”，真正的切点由 `prepareCompaction` 计算。

### 3. 内容点：切点选择会避开“回合中间”

**结论**：`findCutPoint` 从尾部往回累计 token，只允许在合法边界（turn 边界）切，避免把一轮对话拦腰斩断；必要时会保留 turn 前缀摘要。

**源码位置**：[compaction.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/compaction/compaction.ts)

```typescript
export function findCutPoint(entries, startIndex, endIndex, keepRecentTokens): CutPointResult {
	const cutPoints = findValidCutPoints(entries, startIndex, endIndex);
	// 从后往前累计 token，找到第一个超过 keepRecentTokens 的合法切点
	// ...
	const cutEntry = entries[cutIndex];
	const isUserMessage = cutEntry.type === "message" && cutEntry.message.role === "user";
	const turnStartIndex = isUserMessage ? -1 : findTurnStartIndex(entries, cutIndex, startIndex);
	return {
		firstKeptEntryIndex: cutIndex,
		turnStartIndex,
		isSplitTurn: !isUserMessage && turnStartIndex !== -1,
	};
}
```

**讲解**：

- `isSplitTurn` 表示“切点落在某个回合中间”，此时需要把该回合的前半段单独总结，后半段仍保留。
- 这是“不破坏对话连续性”的关键细节：直接按 token 数硬切会让模型丢掉上下文。

### 4. 内容点：压缩结果由“摘要 + retainedTail + 文件操作”组成

**结论**：`prepareCompaction` 算出要总结的历史、保留的尾部、被修改的文件列表，然后交给 `compact` 调用模型生成结构化摘要。

**源码位置**：[compaction.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/compaction/compaction.ts)

```typescript
return ok({
	firstKeptEntryId,
	messagesToSummarize,
	turnPrefixMessages,
	retainedTail,
	isSplitTurn: cutPoint.isSplitTurn,
	tokensBefore,
	previousSummary,
	fileOps,
	settings,
});
```

**讲解**：

- `messagesToSummarize` 是会被总结掉的历史；`retainedTail` 是保留下来的近期消息。
- `fileOps` 记录历史中读过/改过哪些文件，摘要提示词会参考它，避免模型忘记项目状态。
- `tokensBefore` 保留压缩前的用量，用于统计与调试。

### 5. 内容点：摘要提示词是“可读的结构化检查点”

**结论**：压缩不是简单“把旧消息删掉”，而是让模型输出一份带 Goal / Progress / 文件上下文的结构化总结。

**源码位置**：[compaction.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/compaction/compaction.ts)

```text
Create a structured context checkpoint summary that another LLM will use to continue the work.

Use this EXACT format:

## Goal
[What is the user trying to accomplish? ...]

## Progress
### Done
- [x] ...
### In Progress
- [ ] ...
### Blocked
...
```

**讲解**：

- 摘要的受众是“另一个 LLM”，所以格式必须稳定、信息密度高。
- 学习价值：Agent 产品里很多“看不见的 LLM 调用”都有类似精心设计的 prompt。

### 6. 内容点：compaction entry 在会话树里如何生效

**结论**：上下文构建时，`defaultContextEntryTransform` 发现 `compaction` entry 后，会用“摘要 + 保留尾部”替换压缩掉的历史。

**源码位置**：[session.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/session/session.ts)

```typescript
export function defaultContextEntryTransform(pathEntries: SessionTreeEntry[]): SessionTreeEntry[] {
	let compaction: CompactionEntry | null = null;
	for (const entry of pathEntries) {
		if (entry.type === "compaction") compaction = entry;
	}
	if (!compaction) return [...pathEntries];

	const entries: SessionTreeEntry[] = [compaction];
	// 保留 firstKeptEntryId 之后的尾部
	// ...
	return entries;
}
```

**讲解**：

- 历史 entry 仍然存在（不删除数据），只是不再进上下文。
- `retainedTail` 存在 compaction entry 上，保证“最近的内容”不会因为切换分支而丢失。
- 这正是“会话树 + 不可变 entry”设计的好处：压缩只改变投影，不破坏原始记录。

### 7. 内容点：Harness 把压缩挂到循环的钩子上

**结论**：Harness 通过 `transformContext` 钩子介入每一次模型调用前的上下文，让低层循环完全不知道“压缩”的存在。

**源码位置**：[agent-harness.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/agent-harness.ts)

```typescript
transformContext: async (messages) => {
	const result = await this.emitHook({ type: "context", messages: [...messages] });
	return result?.messages ?? messages;
},
```

**讲解**：

- 低层循环只调用 `transformContext`；Harness 在这里让扩展/自身决定是否替换消息。
- 学习价值：把“策略”放在钩子里，核心循环保持简单，是良好的可扩展设计。

## 小结

| 阶段 | 函数 | 产出 |
| --- | --- | --- |
| 估算 | `estimateTokens` | 每条消息 token 数 |
| 判断 | `shouldCompact` | 是否压缩 |
| 切点 | `findCutPoint` | 合法的 firstKept 位置 |
| 准备 | `prepareCompaction` | 摘要输入 + retainedTail + fileOps |
| 生成 | `compact` | 结构化摘要 |
| 生效 | `defaultContextEntryTransform` | 后续上下文只看到摘要+尾部 |

## 练习与思考

1. 给一组消息手工计算 `estimateTokens`，再对照 `shouldCompact` 阈值。
2. 解释为什么 `findCutPoint` 会返回 `isSplitTurn`，并描述该情况下 `turnPrefixMessages` 的作用。
3. 在 `session.ts` 中找 `createCompactionSummaryMessage`，说出它最终变成什么角色消息。

## 延伸问题

- 压缩后的 `previousSummary` 会在下一次压缩时被再次传给模型吗？这会导致“摘要套娃”吗？
- 为什么 `estimateTokens` 对 `image` 用固定字符数而不是 base64 真实长度？
- 如果 Provider 报告的真实 usage 与启发式估算差距很大，Pi 以哪个为准？为什么？
