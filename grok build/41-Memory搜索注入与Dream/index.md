# 41 Memory 搜索、注入与 Dream

Memory 不是把所有旧对话永远塞进当前 prompt，而是把跨 session 的材料切成可索引、可检索、可重新注入的知识。用户入口是 [13-memory.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/13-memory.md)，源码入口是 [memory_state.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/session/memory_state.rs)、[memory_dream.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/session/acp_session_impl/memory_dream.rs) 和 xai-grok-tools/src/implementations/memory。

## 它要解决的约束

普通 chat history 有两个相反的问题：

- 全量保留会迅速挤占 context，下一轮模型变贵、变慢；
- 全量丢弃会让 Agent 忘记项目约定、历史决策和长期目标。

Memory 选择“外置存储 + 搜索时取回”。好处是上下文可控、跨 session 可用、可以按 workspace 分开；代价是检索会漏召回、embedding 可能失败、旧事实会过期，模型看到的不是完整历史。

## SessionMemory 把状态分组

源码摘录：[memory_state.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/session/memory_state.rs) 的 SessionMemory 同时保存存储、首次注入、flush、compaction recovery 和 dream 计数：

~~~rust
pub(crate) struct SessionMemory {
    pub storage: RefCell<Option<crate::session::memory::MemoryStorage>>,
    pub save_on_end: bool,
    pub backend_params: Option<crate::session::memory::MemoryBackendParams>,
    pub initial_injection_config: crate::config::MemoryInitialInjectionConfig,
    pub context_injected: AtomicBool,
    pub flush_config: crate::config::MemoryFlushConfig,
    pub is_flushing: AtomicBool,
    pub last_flush_compaction: AtomicU64,
    pub dream_config: crate::config::MemoryDreamConfig,
}
~~~

把这些字段放进一个 memory state 不是单纯整理代码。它让 /memory on、flush、dream、统计和 session end 使用同一套“是否启用、是否正在写、是否已经注入”的事实，减少散落 bool 产生的矛盾状态。

## 记忆从写入到取回的流程

~~~mermaid
flowchart LR
    S["session turn / session end"] --> F["flush summary or notes"]
    F --> C["chunk files"]
    C --> I["index.sqlite"]
    I --> E{"embedding configured?"}
    E -->|yes| V["embed missing chunks"]
    E -->|no| K["keyword / lexical path"]
    Q["new prompt or memory_search"] --> R["retrieve candidates"]
    V --> R
    K --> R
    R --> M["score, filter, diversify"]
    M --> P["memory context injection"]
    P --> L["model prompt"]
~~~

图的依据是 SessionMemory 的 open_index、reindex_and_embed、memory_dream 的 flush/reindex 路径，以及 memory tool implementation。它把搜索过程压缩成概念步骤；具体 score、字段和阈值要以当前 memory 模块和配置类型为准。

## 首次注入和模型主动搜索不是一回事

首次注入是 session setup 或第一轮时自动做的一次检索；memory_search 是模型在需要时主动调用；compaction recovery 则是在上下文被压缩后再次找回有用材料。三者要分开计数，否则只看到“memory 被使用”却不知道是哪条路径。

一个可理解的伪代码是：

~~~text
if memory_enabled and not context_injected:
    candidates = search(source="injection", prompt_context)
    context = select_with_score_and_diversity(candidates)
    append_system_memory_context(context)
    context_injected = true

if model_calls_memory_search:
    candidates = search(source="tool", query)
    return memory_get_or_search_result(candidates)

if compaction_finished and recovery_enabled:
    run a smaller recovery search
~~~

首次注入的好处是用户不必每次手动提醒；代价是第一轮会增加延迟和 token。为此源码保存了 context_injected，并用 conversation_has_memory_context 做跨 segment 的幂等判断，避免重复注入同一段材料。

## 为什么需要 hybrid、MMR 和衰减

单一向量相似度不适合所有代码问题。文件名、函数名和错误码常常需要 lexical match；自然语言描述又适合 embedding。hybrid search 把两类候选合并，再用 score、最小分数和结果数量控制 prompt 大小。

相似结果如果都来自同一个文件，模型看到的只是重复段落。MMR 或类似的多样性选择会在相关性和新颖性之间取舍。时间衰减可以降低旧记录的权重，但也可能把“长期稳定约定”错误地当成过期信息；实际参数需要用项目数据验证。

| 决策 | 主要收益 | 代价 |
| --- | --- | --- |
| lexical + embedding | 同时照顾符号和语义 | 需要维护两套候选/配置 |
| minimum score | 拦截明显无关内容 | 可能漏掉低相似但关键的事实 |
| MMR/diversity | 减少重复上下文 | 相关度最高的结果可能被换掉 |
| decay/staleness | 降低旧信息干扰 | 稳定规则也可能被降权 |
| workspace index | 项目间不串记忆 | 迁移、清理和路径变化更复杂 |

## flush 和 Dream 的关系

flush 负责把当前 session 的摘要或增量写入 memory；Dream 是在多个 session 日志之上做一次更高层的 consolidation。源码中 dream 有独立 gate、lock、会话列表、模型调用、超时、写回和索引清理，且 subagent 会跳过自动 dream。

~~~text
session end
  -> write session summary
  -> check dream gates and lock
  -> read eligible session logs + existing MEMORY
  -> call dream model
  -> validate/write consolidated memory
  -> reindex new memory
  -> delete stale source paths from index
~~~

Dream 的好处是把很多短日志整理成少量长期事实；代价是又增加一次模型调用和“总结失真”风险。它应该被当作可审计的派生数据，而不是不可质疑的真相。手动 /dream 绕过部分时间或 session gate，也意味着使用它时要更关注成本和重复执行。

## 失败路径与隐私边界

| 失败 | 后果 | 处理 |
| --- | --- | --- |
| embedding provider 不可用 | 新 chunk 没有向量 | 保留 lexical 路径或等待补 embedding |
| index.sqlite 锁冲突 | 搜索/重建失败 | 记录错误，避免破坏源文件 |
| flush 正在进行 | compaction 触发重复写 | is_flushing 锁和 once-per-cycle guard |
| dream 超时 | 当前 consolidation 失败 | 记录 dream_error，不替换旧 memory |
| memory 过期 | 模型得到旧规则 | staleness、decay、人工清理 |
| 项目含密钥/个人信息 | 可能进入索引和 prompt | 在写入前配置排除、审计存储和脱敏 |

Memory 的 workspace 作用域降低了项目之间串数据的概率，但不等于天然隐私隔离；index.sqlite、日志和 embedding provider 都需要按敏感数据处理。

## 本地阅读与验证

~~~bash
rg -n "SessionMemory|open_index|reindex_and_embed|context_injected|memory_search|memory_get|flush|dream|staleness|MMR|embedding" \
  crates/codegen/xai-grok-shell/src/session \
  crates/codegen/xai-grok-tools/src/implementations/memory \
  crates/codegen/xai-codebase-graph

cargo test -p xai-grok-shell memory
cargo test -p xai-grok-tools memory
cargo test -p xai-codebase-graph
~~~

实验时建议创建两条近似但不同的记忆，再观察检索是否重复、是否被过滤，以及关闭 embedding 后 lexical 路径是否仍有结果。不要直接把真实隐私对话拿来做实验。
