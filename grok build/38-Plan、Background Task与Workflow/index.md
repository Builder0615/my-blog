# 38 Plan、Background Task 与 Workflow

Plan mode 和 background task 都是在解决“工作不会在一次模型回复里结束”的问题，但它们控制的对象不同。Plan mode 先约束修改行为，background task 让耗时工作在当前 UI turn 之外继续。对应用户指南是 [19-plan-mode.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/19-plan-mode.md) 和 [20-background-tasks.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/20-background-tasks.md)。

源码阅读入口：

- xai-grok-shell/src/session/acp_session_impl/goal.rs、goal_support.rs：目标/计划和后续推进；
- prompt_queue.rs、notification_drain.rs：提示排队与任务结果注入；
- workflow.rs、tasks_cancel.rs：工作流和取消；
- xai-grok-tools/src/implementations/grok_build/task：任务 coordinator、monitor 和 task output；
- pager 的 views/plan_approval_view.rs、views/tasks_pane.rs、app/dispatch/turn.rs：用户交互。

## Plan mode 真正控制的是“允许做什么”

初学者看到 plan 文件，容易把它理解成一篇给人看的 markdown。实现上它更接近一个持久化状态机：

~~~text
规划中 -> 等待用户审批 -> 批准后执行 -> 反馈/修订 -> 完成
             |                         |
             +----拒绝/取消-------------+
~~~

进入 plan mode 后，Agent 可以读文件、搜索、运行允许的分析工具，却不应该直接执行需要编辑的动作。审批是 ACP reverse request：客户端必须回一个 decision，session 才能释放等待。这样做的好处是把“模型说要改”和“用户授权改”分开；代价是客户端断线、恢复 session 时要重新建立 live waiter。

## plan 审批时序

~~~mermaid
sequenceDiagram
    participant U as 用户
    participant V as TUI/ACP client
    participant A as SessionActor
    participant F as 文件与工具层
    U->>V: 进入 plan mode
    V->>A: prompt
    A->>F: 读取/搜索/分析
    F-->>A: 计划材料
    A->>V: plan update
    A->>V: exit_plan_mode reverse request
    V-->>A: approve / reject / feedback
    alt approve
        A->>A: 开放执行路径
        A->>F: edit / shell / test
    else feedback
        A->>A: 追加反馈并重新规划
    else reject
        A->>A: 结束或等待下一次 prompt
    end
~~~

图依据 goal.rs、goal_support.rs、plan_approval_view.rs 和 plan_mode 测试。它不能证明任何客户端都会画出同样的 UI；ACP client 可以使用不同的外观，但必须处理 reverse request 的协议语义。

## Background task 为什么不等于 tokio::spawn

简单地 tokio::spawn 一个 future，只能解决“当前函数不阻塞”。产品还需要知道：

- task id 和人类可读标题；
- 进度、状态、退出码和输出位置；
- 谁可以 wait、kill、查看 task output；
- 当前 turn 是否要被唤醒；
- session 关闭时 child process 是结束还是继续；
- 多个 client 连接时，哪个客户端收到通知。

因此 Grok Build 用 task coordinator 管理生命周期，再通过 notification drain 把已完成的任务作为输入送回 session。/wait、/kill、/tasks 和 tasks pane 只是这个状态的不同入口。

## Task 与 Agent turn 的边界

源码中的 NotificationPriority 把通知分成 Next 和 Later。这体现了一个很重要的取舍：监控事件可能需要在 tool call 之间尽快进入上下文，普通 bash 完成通知可以等到 turn 结束或 session idle 时再处理。若所有通知都即时注入，模型上下文和用户界面会被噪声打断；若全部延迟，用户会错过失败或完成。

伪代码（说明调度意图）：

~~~text
event = task_coordinator.poll()
if event.priority == Next and session.is_interruptible():
    session.enqueue_system_notification(event)
else:
    session.defer_until_turn_end(event)

if task_finished and user_requested_wait:
    return task_output(task_id)
~~~

## workflow、loop、scheduler 是“再上一层”

| 能力 | 触发 | 持久化/状态重点 |
| --- | --- | --- |
| background command | 当前 turn 创建的命令 | pid、输出、退出码、取消 |
| monitor | 对某个任务持续观察 | monitor id、事件、唤醒策略 |
| /loop | 周期性 prompt/命令 | interval、下一次时间、停止条件 |
| scheduler | 更长期的计划任务 | 任务定义、调度时间、session 关联 |
| workflow | 一组有顺序/条件的动作 | 步骤、输入输出和失败分支 |

这些能力叠加时，不能只看“会不会执行”。还要画出谁拥有取消权、哪个 session 存储状态、输出是插入模型还是仅显示给用户。

## 失败路径

| 失败 | 处理问题 |
| --- | --- |
| plan 文件已批准但 client 消失 | session resume 时重新发审批 reverse request |
| background command 退出但 prompt 已取消 | 是否保留任务、何时 drain 通知 |
| wait 的 task id 不存在 | 返回 typed error，不应让整个 session 崩溃 |
| scheduler 触发时 workspace 被删除 | 任务失败并记录原因，不假装执行 |
| workflow 中间步骤失败 | 保留已完成步骤、停止后续还是按策略重试 |
| Ctrl-C | 可能取消当前 turn，并抑制 queued task wake；不能简单等同 Esc |

## 本地验证

~~~bash
rg -n "PromptMode|exit_plan_mode|plan approval|NotificationPriority|TaskWake|notification_drain|workflow|scheduler|tasks_cancel" \
  crates/codegen/xai-grok-shell/src/session \
  crates/codegen/xai-grok-tools/src/implementations/grok_build/task \
  crates/codegen/xai-grok-pager/src/views \
  crates/codegen/xai-grok-pager/src/app

cargo test -p xai-grok-shell plan
cargo test -p xai-grok-pager tasks
~~~

我在读这部分代码时，会把“计划状态”“后台进程”“通知入队”“UI 控件”分别画成泳道。它们都叫 task/plan，但失败时的责任边界完全不同。

