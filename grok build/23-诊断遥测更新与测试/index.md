# 23 诊断、遥测、更新与测试

一个 Agent 产品要能解释失败、被更新、被监控和被测试。Grok Build 的这些模块不在“核心循环”里，但它们决定了工程是否能被维护。

## doctor、inspect 和配置诊断

CLI 的 `Doctor`、`Inspect`、`Mcp` 等命令是用户和开发者检查运行环境的入口。它们可以暴露 auth、model、MCP、sandbox、session 或 config 问题。读命令分发时，注意这些命令可能在不启动完整 Agent 的情况下工作。

相关源码可以从 [`xai-grok-shell/src/inspect`](https://github.com/xai-org/grok-build/tree/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-shell/src/inspect)、`mcp_doctor.rs`、config tests 和 pager dispatch 开始。

## trace、telemetry 和 crash handler

| 模块 | 关注点 |
| --- | --- |
| `xai-tracing` / `xai-tracing-macros` | span、event 和统一 trace 结构 |
| `xai-grok-telemetry` | 产品 telemetry、采样、隐私和上传 |
| `xai-crash-handler` | panic/crash 收集和退出处理 |
| shell `instrumentation.rs`、upload | turn、trace、usage 和后台上传 |
| `xai-grok-announcements` | 用户可见提示和公告 |

日志不应该混进 stdio ACP 的 stdout；这也是 stdio agent 有兼容 writer 和诊断通道的原因。读遥测时还要确认哪些字段是本地日志，哪些字段会离开机器。

## update 和 leader relaunch

`xai-grok-update` 负责安装/检查新 binary；leader protocol 的 `RelaunchForUpdate` 负责让常驻进程在有界等待后退出，让 client 重新连接新版本。更新不是简单覆盖一个文件，还涉及版本比较、正在运行的 turn、session flush、client reconnect 和失败提示。

## 测试层次

| 测试层 | 可以证明什么 |
| --- | --- |
| 类型/单元测试 | 参数、schema、纯函数和状态转换 |
| crate tests | 一个 crate 的 runtime 行为 |
| session/ACP tests | 命令、事件、reconnect 和 permission 交互 |
| PTY harness | 真终端输入输出、resize、退出恢复 |
| smoke/e2e | 多 crate 组合是否连得起来 |
| 真实模型 eval | prompt、工具和模型之间的行为回归 |

不要用“测试通过”笼统表达证据范围。一个 `cargo test -p xai-grok-config` 不能证明 TUI 或真实模型路径通过；一个单元测试也不能证明 sandbox 在所有平台生效。

## 公开用户指南是测试索引

用户指南中的 `23-dashboard.md`、`24-monitoring-usage.md` 和 `21-terminal-support.md` 能告诉你产品希望支持哪些行为；源码测试再告诉你当前快照实际覆盖到哪里。两者的差异值得记下来。

## 可观测性也有数据边界

trace、debug log、产品 telemetry、crash report 和用户可见 error 不是一份数据的五种格式。前者可能包含 request timing，后者可能需要脱敏；本地日志可以更详细，上传事件应该更克制；stdio 模式还要保证协议 stdout 不被污染。

追一个字段时，我会问：它在哪里创建、在哪个 span 里传播、是否写入 session record、是否上传、采样条件是什么、失败时会不会阻塞主 turn。不要看到 `instrument` 就假设所有信息都能在 dashboard 出现。

## doctor 的价值在于缩短边界

好的 doctor 命令可以在不启动完整 TUI/Agent 的情况下确认 toolchain、config、auth、endpoint、MCP、sandbox 和 workspace。它不应该偷偷执行危险工具，也不应该把 credential 原文打印出来。读它时观察检查项、顺序、退出码、JSON 输出和错误修复建议。

## 更新是一次跨进程状态转换

更新流程至少有发现版本、校验/下载、安装新 binary、通知 leader、等待当前工作到安全点、退出旧进程、重新连接和失败回退。若当前 turn 正在运行，更新策略可能延后、请求 flush、限制重连或保留 session history。

我会把 `RelaunchForUpdate` 当成协议状态来追，而不是一个 `exit(0)`：谁发送、leader 回什么、client 怎样判断新旧版本、断线期间哪些 request 可以重放，都会影响用户是否丢工作。

## 测试要写“它排除了什么”

读测试时可以给每个 test 加一句证据说明：它证明了某个纯函数、一个 actor 命令、一个 protocol frame、一个真实 PTY 行为，还是多 crate 的组合。没有网络/模型的测试不能证明远端 endpoint；没有 OS sandbox 的测试不能证明隔离；没有真实 terminal 的 snapshot 不能证明键盘恢复。

这套证据习惯也适用于学习笔记：记录“测试名—准备条件—断言—未覆盖边界”，比只写“测试通过”更能帮助下一次上游同步。

## 小实验

```bash
rg -n "doctor|inspect|telemetry|trace|crash|update|RelaunchForUpdate|pty|snapshot|e2e" \
  crates/codegen/xai-grok-pager-bin \
  crates/codegen/xai-grok-shell \
  crates/codegen/xai-grok-update \
  crates/codegen/xai-grok-pager-pty-harness
```

从一个失败测试开始，记录它能排除哪一类错误、不能排除哪一类错误。这是建立“证据边界”的最好练习。
