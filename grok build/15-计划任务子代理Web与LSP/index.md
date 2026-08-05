# 15 计划、任务、子代理、Web 与 LSP

工具系统不只有文件和 shell。Grok Build 还把“把工作拆出去”“在后台跑”“用计划约束编辑”“查网页”“问语言服务器”这些能力也纳入 Agent runtime。它们共同点是：调用之后不一定马上得到一个短字符串。

## plan、todo 和 goal 不是同一个东西

| 能力 | 更像什么 | 源码线索 |
| --- | --- | --- |
| plan mode | 编辑前的状态门禁和计划文件流程 | `enter_plan_mode`、`exit_plan_mode`、`session/plan_mode.rs` |
| todo | 当前任务的可见清单 | `implementations/grok_build/todo`、shell todo |
| goal | 跨多轮判断是否完成的目标状态 | `update_goal`、`goal_*` 模块 |
| workflow | 可复用的多步流程 | `workflow` tool、`xai-workflow` |

不要把它们都写成“模型的计划”。plan mode 可能控制能不能编辑，todo 是状态展示，goal 有 stop/evaluate/orchestrate 的组件，workflow 还涉及外部定义和调度。

## background task 的生命周期

`task`、`task_output`、`wait_tasks`、`kill_task`、`monitor` 和 scheduler 类工具说明：一个 tool call 可能启动一个比当前 turn 更长的任务。任务状态要保存在 subagent/task registry，输出要能被后续调用查询，取消还要能传到子进程或子 session。

阅读顺序可以是：

1. 找 task 参数 schema；
2. 找任务对象由谁保存；
3. 找 output 如何流入 session notification；
4. 找 wait/kill 如何处理已经完成、正在运行和不存在的 task；
5. 找 TUI 和 headless 各自如何展示。

## 子代理不是一次普通函数调用

shell 的 [`agent/subagent`](https://github.com/xai-org/grok-build/tree/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/agent/subagent) 和 pager 的 `app/subagent.rs` 会涉及 child session、agent persona、能力模式、usage 汇总和父子路由。子代理可能有自己的 history、toolset、model 和生命周期，但结果要回到父 session。

这里的核心问题是 ownership：谁能取消子代理？谁支付 token usage？子代理的 tool permission 是继承、复制还是重新询问？只有看到 `subagent` tests 和 session routing 才能确认，不要用“并行调用模型”概括。

## Web、image/video 和 LSP

| 工具类别 | 代码线索 | 特别要注意 |
| --- | --- | --- |
| `web_search` / `web_fetch` | `implementations/grok_build/web_*`、工具 resources | 网络、域名权限、内容截断 |
| image/video | `image_edit`、`image_gen`、`video_gen`、session image normalize | 文件路径、异步生成、渲染能力 |
| LSP | `implementations/grok_build/lsp`、`implementations/lsp` | server 生命周期、workspace root、诊断格式 |
| workflow | `implementations/grok_build/workflow`、`xai-workflow` | 多步状态和可恢复性 |

外部服务的返回内容不是天然可信。它们仍然要经过输出限制、权限、错误转换和上下文预算，不能直接把网页或 LSP 结果无限塞进模型。

## 计划门禁和执行顺序

tool_calls 执行器会先 prepare 一批调用，再决定哪些可以运行；`exit_plan_mode` 需要在 body 调用之后处理。这说明“模型一次返回多个调用”不等于它们拥有相同的顺序和权限。

## 这些能力为什么不能都叫“工具”

它们都可能表现为 model tool，但生命周期不同：

| 能力 | 主要状态 | 结果怎样回来 |
| --- | --- | --- |
| plan/todo | 当前目标、步骤、完成状态 | session event、提示词或文件记录 |
| background task | task id、进程、输出、退出状态 | task registry + poll/wait |
| subagent | child session、persona、history、usage | 父 session 的 tool result/事件 |
| web/image/video | 外部请求、响应、媒体或引用 | 受限内容进入模型/界面 |
| LSP | server、workspace、诊断/符号请求 | 结构化编辑器结果 |
| workflow | 多步状态、重试、恢复、触发器 | 父流程或 session 更新 |

把它们混成同一种 tool，会遗漏“谁拥有状态”和“谁负责回收”。一个 `task` 返回 id 后，当前 turn 可能结束但后台进程仍在；一个 subagent 返回前，父 session 可能需要继续接收 interjection；一个 LSP server 可能是长生命周期资源，而不是每次请求都启动。

## plan mode 是执行门，不是装饰性提示

进入 plan mode 后，模型可能只能读取和规划，写操作要等用户确认或 `exit_plan_mode`。这会改变 tool registry 的可用性、permission decision 和 turn 的继续条件。读它时要分别看：模式状态存在哪里、哪些工具被 gate、确认结果如何记录、退出失败是否恢复普通模式。

## background task 的两个时间轴

后台任务有“启动时间轴”和“观察时间轴”：

```text
spawn task -> return task id -> parent turn may continue/end
                         |
                         +-> task_output / wait_tasks / kill_task
                         +-> exit event / cleanup / final record
```

如果只追 `spawn`，会以为任务已经完成；如果只追 `wait`，又看不到它如何获得权限和 workspace。排查泄漏时，重点看 task registry 的删除条件、parent session 关闭时是否 kill，以及输出是否有上限。

## subagent 是一个嵌套的产品边界

subagent 可能拥有自己的 model、toolset、prompt、history 和 session lifecycle。父 Agent 看到的是调用结果，不应直接假设子 Agent 的所有中间事件都进入父 history。读 `agent/subagent` 时要找 child session 的创建、能力限制、usage 汇总、取消传播和结果压缩。

## 外部服务必须被当成不可信输入

web、image、video、LSP 和 MCP 返回的内容都可能很大、格式异常或包含指令性文本。工具实现需要做大小限制、错误转换、权限/域名校验和上下文清理；模型 prompt 也要区分“外部资料”与“系统指令”。这不是为了把工具做得保守，而是为了让失败可解释、上下文可控。

## 小实验

```bash
rg -n "task_output|wait_tasks|kill_task|spawn_subagent|enter_plan_mode|exit_plan_mode|workflow|lsp" \
  crates/codegen/xai-grok-tools/src/implementations \
  crates/codegen/xai-grok-shell/src/session \
  crates/codegen/xai-grok-shell/src/agent/subagent
```

选 `task` 和 `spawn_subagent` 做对比：一个返回什么句柄，结果如何查询，父 session 在等待期间能不能接收用户 interjection？
