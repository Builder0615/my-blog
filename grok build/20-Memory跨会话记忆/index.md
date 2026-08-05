# 20 Memory：跨会话记忆

Memory 和当前 conversation 不一样。conversation 记录这次 session 的对话；memory 让未来的 session 可以检索过去保存的知识。它既涉及文件布局，也涉及索引、搜索、凭据和后台 flush。

## 默认存在哪里

[`xai-grok-memory/src/lib.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-memory/src/lib.rs) 说明了跨 session Markdown memory 的布局：

```text
~/.grok/memory/MEMORY.md
~/.grok/memory/{blake3(cwd)[..16]}/MEMORY.md
~/.grok/memory/{hash}/sessions/YYYY-MM-DD-{slug}-{sid8}.md
```

它由 `--experimental-memory` 或 `GROK_MEMORY=1` 开启。hash 的作用是把不同工作目录的项目记忆隔开；session 文件则让某次对话可以被后续搜索。

## 搜索 backend

backend 在 [`xai-grok-memory/src/backend.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-memory/src/backend.rs)：

- SQLite FTS5 提供关键词检索；
- 可选的 `sqlite-vec` 提供向量 KNN；
- rusqlite `!Sync` 影响连接的打开方式；
- dirty-file watcher 触发重新索引；
- embedding credentials 按 endpoint scope 管理，并有 fail-closed 行为。

所以 memory search 不是一个永远在线的全局向量数据库。每次查询、索引和 embedding 失败都可能有独立路径。

## memory 如何进入 Agent

当前代码至少有几条入口：

| 路径 | 用途 |
| --- | --- |
| `memory_search` / `memory_get` tool | 模型主动查询或读取记忆 |
| initial/recovery injection | session 新建或恢复时提供相关内容 |
| `/flush` 或 session flush | 把当前对话中的知识写回 memory |
| `/dream` / idle timer | 在空闲时整理或生成跨 session 记忆 |
| memory reminder | 在 request context 中提示模型如何使用 memory |

因此不能只看 base prompt 就断言“memory 永远注入 system prompt”。要同时读 [`session/memory_state.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/session/memory_state.rs)、memory tool 和 prompt tests。

## memory 的状态机

session memory state 会记录 storage、backend params、initial injection、flush config、reindex、counters 和 dream config。run loop 还会启动 idle memory flush/dream timers，这些后台动作和主 turn 可能并行。

并行带来的问题是：

- flush 是否会阻塞下一次 prompt？
- reindex 期间 query 读到旧索引还是等待？
- session 结束时怎样等待 pending memory write？
- embedding 失败时是否保留纯文本 memory？

这几个问题都不能只靠用户指南的开关说明回答。

## 文件、索引和 tool 是三套状态

memory 的 Markdown 文件是人可以检查的事实来源，SQLite/FTS/向量索引是查询加速结构，memory tool 是模型与它交互的受控入口。索引损坏不一定意味着文件丢失；文件写成功也不代表马上能搜到；tool 返回结果还要经过大小限制和权限。

```text
memory file --(watch/reindex)--> search index
     ^                              |
     |                              v
  flush/dream <--- memory tool <--- query
```

排查“memory 没生效”时，要分别检查文件是否生成、索引是否包含、查询是否命中、结果是否被注入 request。跳过其中一层，容易把索引延迟误判成写入失败。

## FTS5 和向量检索适合不同问题

FTS5 适合精确词、路径、符号和错误码；向量 KNN 适合措辞不同但语义相近的记忆。向量检索还依赖 embedding endpoint、模型、凭据和索引版本，失败时可能要退回关键词搜索或返回明确不可用。

所以 memory search 的结果排序、去重和 fallback 值得读测试。一个“搜不到”可能是词不匹配、embedding 未生成、项目 hash 不同、索引过期或 memory scope 被限制。

## flush 和 dream 的语义不同

`flush` 更像把当前 session 中应该保留的知识写出；`dream`/idle timer 更像在没有用户输入时整理、总结或生成候选。两者都可能触发模型调用和文件写入，但生命周期、取消条件和用户可见性不同。

学习时我会问：session 关闭是否等待 flush？用户新 prompt 到来时 dream 会取消吗？dream 生成的内容是否需要确认？embedding 失败后纯文本能否保留？这些问题决定 memory 是否会在后台悄悄改变用户目录。

## scope 和隐私边界

全局 memory、项目 memory、session memory 和 embedding 服务可能有不同的路径与权限。项目 hash 用来隔离目录，不代表内容天然脱敏；memory search 返回的历史片段也可能包含敏感代码。产品开关、文件权限、endpoint scope 和日志脱敏要一起看。

## memory 和隐私

memory 会把内容写到 `~/.grok/memory`，因此它和当前项目、用户目录、embedding endpoint、权限和导出路径有关。学习时不要把个人 token、生产凭据或敏感代码放进真实 memory 实验。

## 小实验

```bash
rg -n "GROK_MEMORY|experimental-memory|memory_search|memory_get|flush|dream|reindex|MEMORY.md" \
  crates/codegen/xai-grok-memory \
  crates/codegen/xai-grok-shell/src/session \
  crates/codegen/xai-grok-tools/src/implementations
```

画两条线：一条是“模型查询旧 memory”，一条是“session 把新知识写回 memory”。分别标出文件、索引、tool 和后台 timer，别把读写流程画成一次操作。
