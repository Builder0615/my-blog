# 37 Subagent、Persona 与隔离

Subagent 是“由父 Agent 委托出去的一次独立工作”，不是把 prompt 递归地塞回当前对话。对应用户指南是 [16-subagents.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/16-subagents.md)。源码可以从 [agent/subagent/mod.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/agent/subagent/mod.rs)、[handle_request.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/agent/subagent/handle_request.rs) 和 [subagent_coordinator.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/agent/mvp_agent/subagent_coordinator.rs) 开始。

## Agent、Persona、Subagent 是三个维度

| 名称 | 作用 | 是否一定创建 child session |
| --- | --- | --- |
| Agent definition | 一套模型、系统提示词、工具和 capability 的定义 | 被选中执行时通常会建立运行实例 |
| Persona | 对 Agent 的语气、职责、提示词或颜色的包装 | 不等于隔离进程 |
| Subagent | 父 session 委托出的执行实例 | 是，拥有 child 生命周期和结果回传 |

内置的 general-purpose、explore、plan 是产品层角色；用户定义的 .md agent 则通过 discovery 进入目录。不要因为文件叫 persona，就认为它天然拥有独立 sandbox；真正的隔离要看 child session 的 workspace、permission 和 capability。

## 为什么不能只用一个共享对话

一个父 Agent 同时让三个“专家”工作时，共享历史会带来三个问题：

1. 探索员读了大量文件，会挤占父模型的 context；
2. 只读任务可能误拿到父 Agent 的写权限；
3. 子任务的中间步骤、取消和超时会污染父 turn 的完成语义。

独立 child session 的收益是 context、取消、模型和工具配置可以分开；代价是要维护 parent-child 关系、结果摘要、并发上限、深度限制和更多持久化文件。另一种方案是把多个角色串行写进同一个 prompt，代码更少，但无法真正隔离权限，也很难让用户单独查看或停止其中一个任务。

## 源码中已经把 child 的上下文收集成结构

源码摘录：[agent/subagent/mod.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/agent/subagent/mod.rs) 明确记录 child 初始上下文来自哪里：

~~~rust
pub(crate) enum InitialContextSource {
    New,
    Forked,
    Resumed,
}
~~~

同一个文件的 SubagentSpawnContext 还会携带 LSP、workspace operations、permission handle、remote settings、auto-compact tier 等信息。这个结构的意义是：协调器不需要拿着整个 MvpAgent 进去创建 child；它只拿到创建 child 所需的依赖，边界更清楚，也更容易测试继承关系。

## child 的数据流

~~~mermaid
flowchart LR
    P["Parent Session"] --> R["Subagent request"]
    R --> C["Task coordinator actor"]
    C --> S["Child session setup"]
    S --> W{"worktree / shared workspace"}
    W -->|success| WT["isolated worktree"]
    W -->|fallback| SH["shared workspace + warning"]
    S --> T["child tool context"]
    T --> L["inherited LSP / fs / terminal / env"]
    C --> O["child progress + completion"]
    O --> P
~~~

图的依据是 subagent_coordinator.rs 的 child lifecycle、handle_request.rs 的 workspace/worktree 分支，以及 SubagentSpawnContext 的依赖字段。它表示“可能的路径”，不是说每个 child 都会建立 worktree；配置、VCS 和创建失败会决定实际分支。

## 一个 child 从 spawn 到完成会经历什么

伪代码（压缩真实调用链，不是可编译源码）：

~~~text
parent_request = parse_task_request(input)
limits = coordinator.admit(parent_request.depth, parent_request.capability)
ctx = build_subagent_spawn_context(parent)
child = run_shell_child(ctx, initial_context_source)
child_session = child.setup(model, prompt, workspace, permission)
coordinator.mark_started(child.id)
while child.has_work():
    update = child.next_update()
    publish_to_parent(update)
    if cancelled_or_timeout(): child.cancel()
result = child.last_assistant_message_or_error()
coordinator.mark_completed(child.id, result)
parent_session.receive_child_result(result)
~~~

“返回末尾一条 assistant message”适合把结果交还给父模型，但会丢掉中间工具细节。Grok Build 同时保留 progress/update 供 UI 和 dashboard 展示；父模型拿到的摘要和用户看到的完整 child 轨迹并不是同一份数据。

## capability mode 和权限不要只看名称

| 模式 | 直观含义 | 需要关注的额外边界 |
| --- | --- | --- |
| read-only | 可以读和分析 | LSP、搜索、部分诊断仍可能访问外部进程 |
| read-write | 可以改工作区 | 是否隔离 worktree、是否能写到 cwd |
| execute | 可以运行命令 | shell 环境和 sandbox profile 仍生效 |
| all | 组合能力 | 不会自动绕过组织策略或 yolo 禁用 |

父 Agent 的 MCP、LSP、workspace handle 可以继承，但“继承 handle”不等于“继承所有权限”。代码里还要检查 PermissionMode、sandbox profile、tool overrides 和组织策略。

## worktree 隔离的好处与退化路径

独立 worktree 让多个 child 可以同时编辑不同副本，父目录不马上被半成品污染；同时也有成本：创建、清理、分支命名、未提交变更和路径回收都需要处理。源码在 handle_request.rs 中对“创建失败、恢复时目录消失、没有 snapshot”准备了 fallback 到共享 workspace 的路径。

所以读者做实验时要观察三件事：

- child 的 cwd 是否真的变化；
- git branch/worktree 状态是否变化；
- fallback 是否产生可见 warning，而不是静默假装隔离。

## 取消、超时和嵌套深度

child 取消至少有两层：协调器停止 child 工作，父 session 决定是否把取消原因展示或写入历史。后台任务、子 child 和当前 turn 不一定共用同一个取消策略。SessionCommand::Cancel、CancelOptions 和 task coordinator 是较好的交叉入口。

嵌套深度限制不是为了限制“聪明程度”，而是为了防止树形任务无限展开、资源失控和结果难以回收。独立 child 的另一个代价是 token、进程和 MCP 连接可能按树增长；实际设计通常需要限制深度、并发、时间和输出大小。

## 本地验证

~~~bash
rg -n "InitialContextSource|SubagentSpawnContext|run_shell_child|ChildCompletion|ChildControl|worktree|fallback|depth" \
  crates/codegen/xai-grok-shell/src/agent/subagent \
  crates/codegen/xai-grok-shell/src/agent/mvp_agent \
  crates/codegen/xai-grok-tools/src/implementations/grok_build/task

cargo test -p xai-grok-shell subagent
cargo test -p xai-grok-pager subagents
~~~

阅读时我会把“角色定义”“child runtime”“任务协调”“UI 展示”分开，否则很容易把一个 catalog 文件误认为是完整的并发实现。
