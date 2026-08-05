# 44 Dashboard、Usage 与 OTEL 监控

Dashboard 解决“我同时开了几个 Agent，它们分别在做什么”的问题；usage 和 telemetry 解决“这次运行消耗了什么、哪里慢、错误怎样聚合”的问题。它们都观察 session，却不能把 UI 状态和遥测事件混成一个数据源。

用户入口是 [23-dashboard.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/23-dashboard.md) 和 [24-monitoring-usage.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/24-monitoring-usage.md)。

源码入口：

- xai-grok-pager/src/views/dashboard/{state,row,layout,render,peek}.rs；
- xai-grok-pager/src/app/dispatch/dashboard.rs、dashboard_telemetry.rs；
- xai-grok-shell/src/agent/session_metrics.rs、upload/trace.rs；
- [xai-grok-telemetry/src/events.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-telemetry/src/events.rs)；
- xai-grok-telemetry/src/external/{config,emit,providers,redact,schema,truncate}.rs；
- xai-grok-telemetry/src/otel_layer 和 session_metrics.rs。

## Dashboard 的数据来自哪里

Dashboard row 主要由 app.agents 的 top-level agents 和 subagents 构建。源码注释说明 rows 每个 render frame 重建，不缓存整份列表；排序 key 会按状态和 last_change_at 重新计算。这个选择适合单个 pager 里数量不大的 agent：状态不会因为缓存忘记刷新，代码也保持纯函数式；代价是 agent 数量巨大时每帧重建的成本会增长。

Dashboard 的 persisted state 只保存用户布局偏好，例如 pin、reorder、grouping、filter 和 enable；运行状态仍来自 live roster。这样重启后能保留“我喜欢怎样看”，却不会把已经退出的进程伪装成仍在工作。

## 打开 Dashboard 的门

源码中 dashboard_enabled 先看环境变量 GROK_AGENT_DASHBOARD=0，再看持久化配置；打开动作还检查 auth_state 和 folder trust。原因是打开 dashboard 会切换视觉焦点，若用户还没有登录或尚未回答目录信任问题，应该停留在原来的问题上，而不是让问题被一个新 view 隐藏。

源码摘录：[views/dashboard/mod.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/src/views/dashboard/mod.rs)：

~~~rust
pub fn dashboard_enabled() -> bool {
    if std::env::var_os("GROK_AGENT_DASHBOARD")
        .as_deref()
        .is_some_and(|v| v == std::ffi::OsStr::new("0"))
    {
        return false;
    }
    state::load_persisted_enabled().unwrap_or(true)
}
~~~

这个小函数背后是配置优先级和交互安全：env override 适合临时禁用，persisted setting 适合长期偏好，auth/trust gate 则属于 runtime 前置条件。

## Dashboard、session 和 ACP roster

~~~mermaid
flowchart LR
    A["local app.agents"] --> R["build rows / classify state"]
    B["leader or ACP roster"] --> R
    R --> F["filter / group / sort"]
    F --> V["dashboard render + peek"]
    V --> D["attach / dispatch / focus"]
    D --> S["session prompt or session switch"]
    S --> A
    S --> B
~~~

图依据 views/dashboard/row.rs、state.rs、app/dispatch/dashboard.rs 以及 leader roster 轮询路径。它表达的是“展示状态和操作入口”的循环；具体 client 是否能 attach、是否有 leader，需要看运行模式和 ACP capability。

## Usage 不是一个数字

一次 turn 的使用量至少要区分：

| 维度 | 例子 |
| --- | --- |
| token | input、output、cached、reasoning 或 provider 计费字段 |
| 时间 | queue、model stream、tool、MCP、sandbox |
| 资源 | child process、worktree、embedding、媒体上传 |
| 结果 | completed、cancelled、auth error、rate limit、max turns |
| 归属 | parent session、subagent、background task |

如果只显示总 token，用户无法解释为什么一个只读任务很贵，也无法判断费用是父 Agent 还是 child 产生。session metrics 和 trace metadata 需要携带 session/turn/subagent 关联，但又要避免把 prompt 原文直接上传。

## OTEL / external telemetry 的链路

伪代码（说明数据边界）：

~~~text
event = create_typed_event(session, turn, outcome, usage)
event = redact_secrets(event)
event = truncate_large_fields(event)
if external_telemetry_enabled:
    provider = resolve_provider(config)
    emit(provider, schema_versioned(event))
else:
    write_local_diagnostic(event)
~~~

源码的 external 目录把 config、emit、providers、redact、schema 和 truncate 分开，是一种“先整形，再传输”的设计。收益是 provider adapter 不必各自实现脱敏；代价是每个新增字段都要检查 schema、redaction 和 size limit 是否同步。

## 为什么 UI 状态不能直接当遥测

Dashboard 的 row 可能是一帧渲染出来的临时分类；telemetry 是跨进程、跨 session 的事件序列。UI 可以显示“Working”，但事件上传可能延迟、被禁用或失败；telemetry 里出现 turn_completed，也不能反推出用户当前是否还开着 dashboard。

两者的交集应该是稳定 id、状态枚举和时间戳，而不是共享整个 struct。这样更容易处理断线、重连、脱敏和版本演进。

## 失败路径

| 现象 | 可能原因 |
| --- | --- |
| Dashboard 空白 | auth/trust gate、dashboard disabled、roster 尚未到达 |
| row 状态过期 | live notification 丢失、轮询延迟、session 已退出 |
| attach 失败 | session id 不在当前 client、leader 连接断开、权限不足 |
| usage 缺字段 | provider 不返回某类 token、子代理归属没有传播 |
| OTLP 上传失败 | endpoint、证书、网络、collector backpressure |
| telemetry 包含敏感文本 | redact/truncate schema 漏配，应立即检查事件字段 |
| trace 和 UI 对不上 | 两者观察窗口和事件完成时机不同 |

## 本地验证

~~~bash
rg -n "dashboard_enabled|build_rows|PersistedDashboard|RowState|roster|attach|usage|OTLP|redact|truncate|schema_version|session_metrics" \
  crates/codegen/xai-grok-pager/src/views/dashboard \
  crates/codegen/xai-grok-pager/src/app \
  crates/codegen/xai-grok-telemetry/src \
  crates/codegen/xai-grok-shell/src/agent

cargo test -p xai-grok-pager dashboard
cargo test -p xai-grok-telemetry
cargo test -p xai-grok-shell telemetry
~~~

做监控实验时同时记录本地 UI、session 文件、RUST_LOG 和 collector 输出。只截一张 Dashboard 图片，无法证明事件已经上传或脱敏成功。

