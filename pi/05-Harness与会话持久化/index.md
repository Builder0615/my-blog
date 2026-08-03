# 05 Harness 与会话持久化

第三章的 `agentLoop` 是无状态的：一次调用结束，状态就没了。但一个能长期使用的 Agent 需要记住会话、管理配置、处理并发，还要把每一步写进磁盘。这一章看 Pi 怎么用 `AgentHarness` 把这些能力补上。

## 先看哪些文件

| 文件 | 作用 |
| --- | --- |
| [packages/agent/src/harness/agent-harness.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/agent-harness.ts) | 编排层：phase、turn 快照、持久化、hook |
| [packages/agent/src/harness/session/session.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/session/session.ts) | 会话树、上下文构建 |
| [packages/agent/src/harness/session/jsonl-repo.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/session/jsonl-repo.ts) | JSONL 会话存储 |
| [packages/agent/src/harness/session/repository.ts](https://github.com/earendil-works/pi/blob/main/packages/agent/src/harness/session/repository.ts) | `SessionRepository` 接口与 fork 语义 |
| [packages/storage/sqlite-node/src/sqlite/migrations/001_initial.sql](https://github.com/earendil-works/pi/blob/main/packages/storage/sqlite-node/src/sqlite/migrations/001_initial.sql) | SQLite 初始表结构 |
| [packages/agent/docs/agent-harness.md](https://github.com/earendil-works/pi/blob/main/packages/agent/docs/agent-harness.md) | Harness 生命周期设计文档 |

## 为什么低层循环之上还需要一个 Harness

`AgentHarness` 补上配置管理、会话持久化、资源解析、操作锁、hook 这些长任务需要的能力：

```typescript
export class AgentHarness<...> {
	private session: Session;
	readonly models: Models;
	private phase: AgentHarnessPhase = "idle";
	private activeAbortController?: AbortController;
	private readonly activeTasks = new Map<Promise<void>, TrackedTaskKind>();
	private pendingSessionWrites: PendingSessionWrite[] = [];
	private model: Model<any>;
	private thinkingLevel: ThinkingLevel;
	private systemPrompt: AgentHarnessSystemPrompt<...> | undefined;
	private resources: AgentHarnessResources<...>;
	private tools = new Map<string, TTool>();
	// ...
}
```

- Harness 持有 `Session`（持久化会话）、`Models`（模型注册表）、工具表、资源和各种队列。
- 低层循环只负责转起来，Harness 负责转完之后状态写到哪里、下一轮用什么配置。
- 执行引擎和宿主应用分开，是 Agent 框架很常见的分层方式。

## phase 状态机是并发安全的基石

Harness 用 `AgentHarnessPhase` 表达当前是否繁忙。`prompt`、`compact`、`navigateTree` 这类结构性操作只允许在 `idle` 时开始，否则直接抛 `"busy"`：

```typescript
if (this.phase !== "idle") throw new AgentHarnessError("busy", "AgentHarness is busy");
this.phase = "turn";
try {
	const turnState = await this.createTurnState();
	return await this.executeTurn(turnState, text, operation.signal, options);
} finally {
	this.phase = "idle";
}
```

- 在第一个 `await` 之前同步把 phase 置为 `"turn"`，两个 prompt 不可能并发进入。
- `finally` 保证无论成功失败都回到 `idle`。
- `compact()` 使用 `"compaction"`、`navigateTree()` 使用 `"branch_summary"`，外部可以区分当前在做什么。

## turn 快照是“本轮请求的完整配置”

每次 turn 开始，Harness 先冻结一份快照：消息、资源、system prompt、model、thinking level、工具、stream options。后续逻辑都基于这份快照，避免运行中配置漂移：

```typescript
private async createTurnState(): Promise<AgentHarnessTurnState<...>> {
	const context = await this.session.buildContext();
	const resources = this.getResources();
	const sessionMetadata = await this.session.getMetadata();
	const toolContext = await this.resolveToolContext();
	const tools = [...this.tools.values()];
	const activeTools = this.activeToolNames.map((name) => this.tools.get(name)).filter(...);
	let systemPrompt = "You are a helpful assistant.";
	if (typeof this.systemPrompt === "string") {
		systemPrompt = this.systemPrompt;
	} else if (this.systemPrompt) {
		systemPrompt = await this.systemPrompt({ session, model, thinkingLevel, activeTools, resources });
	}
	return { messages: context.messages, resources, toolContext, systemPrompt, model, thinkingLevel, tools, activeTools };
}
```

- `session.buildContext()` 从会话树生成本轮消息；资源、工具上下文、系统提示都在这时解析。
- 运行中调用 `setModel()` 等 setter 只影响下一轮快照，不污染当前正在进行的请求。

## save point 在 turn 结束后刷新状态并落盘

`prepareNextTurn` 在下一轮开始前 flush 待写会话数据并重新创建 turn 快照；`turn_end` 事件处理完成后发射 `save_point`：

```typescript
prepareNextTurn: async () => {
	await this.flushPendingSessionWrites();
	const nextTurnState = await this.createTurnState();
	setTurnState(nextTurnState);
	return {
		context: this.createContext(nextTurnState),
		model: nextTurnState.model,
		thinkingLevel: nextTurnState.thinkingLevel,
	};
},
```

- 用户运行时切模型或调 thinking level，也能在下一轮生效，不打断当前流。
- `handleAgentEvent` 对 `message_end` 立即写一条消息；对 `turn_end` flush 所有 pending 写并发射 `save_point`；对 `agent_end` 收尾并回到 `idle`。

## 会话是一棵带 leaf 指针的树

`Session` 不把消息存成列表，而是存成带 parent 的 entry 树，并用 `leafId` 指向当前分支。fork、分支切换都建立在它上面：

```typescript
export function buildSessionContext(
	pathEntries: readonly SessionTreeEntry[],
	options: SessionContextBuildOptions = {},
): SessionContext {
	const state = deriveSessionContextState(pathEntries);
	const contextEntries = buildContextEntries(pathEntries, options);
	const messages = contextEntries.flatMap((entry, index) =>
		sessionEntryToContextMessages(entry, index, contextEntries, options),
	);
	return { ...state, messages };
}
```

- entry 类型包括 `message`、`custom_message`、`compaction`、`branch_summary`、`model_change`、`leaf` 等。
- 构建上下文时沿当前 branch 读取 `pathEntries`，再做转换与投影，不是直接翻全部历史。
- 切换 leaf 等于换一条对话路径，这就是分支会话的基础。

## 存储后端抽象与 JSONL 实现

`SessionRepository` 定义 create/open/list/delete/fork 五件套，默认实现把每个会话写成一个版本化的 JSONL 文件：

```typescript
export interface SessionRepository<...> extends AsyncDisposable {
	create(options: TCreateOptions): Promise<Session<TMetadata>>;
	open(metadata: TMetadata): Promise<Session<TMetadata>>;
	list(options?: TListOptions): Promise<TMetadata[]>;
	delete(metadata: TMetadata): Promise<void>;
	fork(source: TMetadata, options: SessionForkOptions & TCreateOptions): Promise<Session<TMetadata>>;
}
```

- JSONL 文件第一行是 `SessionHeader`（type/version/id/cwd），后面每行一个 entry；`parseHeader` 会对版本做严格校验。
- `KeyedOperationQueue` 限制同一会话的并发写操作，避免 append 竞争。
- 持久化被抽象成接口后，就能无痛替换成 SQLite 等后端。

## SQLite 后端把树关系物化成表

SQLite 后端用 `session_entries` 存原始 entry，用 `branch_entries` 和 `session_materialized` 支撑分支查询与物化视图：

```sql
CREATE TABLE IF NOT EXISTS session_entries (
	session_id TEXT NOT NULL,
	id TEXT NOT NULL,
	entry_seq INTEGER NOT NULL,
	parent_id TEXT NULL,
	type TEXT NOT NULL,
	timestamp TEXT NOT NULL,
	payload TEXT NOT NULL,
	PRIMARY KEY (session_id, id)
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_session_entries_session_seq ON session_entries(session_id, entry_seq);
```

- `parent_id` 保存树结构；`entry_seq` 保证同一会话内顺序稳定。
- `branch_entries` 把“分支 → entry 顺序”显式物化，让按分支读取变成一次索引查询。
- 对比 JSONL：JSONL 简单直观，适合阅读和调试；SQLite 适合大量会话的检索与分支操作。

## 概念速查

| 概念 | 作用 |
| --- | --- |
| phase | 防止结构性操作并发 |
| turn 快照 | 冻结本轮完整配置 |
| save point | turn 结束后落盘并刷新 |
| 会话树 + leaf | 支持分支和恢复 |
| SessionRepository | 存储后端抽象 |
| JSONL / SQLite | 两种可替换后端 |

## 动手验证

1. 读 `handleAgentEvent`，画出 `message_end → turn_end → agent_end` 时哪些写会落盘。
2. 对比 `jsonl-repo.ts` 的 `parseHeader` 与 SQLite 迁移文件，各写一条优缺点。
3. 用 `SessionRepository.fork` 的语义解释“在某个用户消息之前 fork”意味着什么。

## 我还没想明白的问题

- 为什么 Harness 选择 settlement 时 flush pending writes，而不是每次事件都同步写？
- 会话树里的 `model_change` 和 `thinking_level_change` entry 为什么也要参与上下文构建？
- 如果一次运行中途进程崩溃，JSONL 和 SQLite 哪个更可能丢失最后几条消息？为什么？
