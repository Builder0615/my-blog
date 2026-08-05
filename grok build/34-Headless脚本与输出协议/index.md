# 34 Headless 脚本与输出协议

Headless 不是“把 TUI 隐藏起来”。它有自己的 stdout/stderr contract、输出 reducer、session flags、tool filtering、permission rules、usage/cost 语义和退出码。用户入口是 [`14-headless-mode.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/14-headless-mode.md)，源码看 `xai-grok-pager/src/headless.rs`、`headless/cli.rs`、`headless/reducer`、`headless/ext_protocol.rs` 与 `xai-grok-pager-bin/src/main.rs`。

## headless 的最小调用

```bash
grok -p "检查当前项目的测试入口" \
  --output-format json \
  --disallowed-tools "search_replace,run_terminal_cmd"
```

`-p/--single`、`--prompt-json`、`--prompt-file` 都会进入 headless。进程完成 response 后退出；是否允许工具、是否自动批准、是否限制 max turns、是否使用 sandbox 都要显式看参数和 config。

## 四种输出格式不能混着解析

| 格式 | 形态 | 适合 |
| --- | --- | --- |
| `plain` | 最终可读文本 | 人工查看、简单 pipe |
| `json` | 一次性 JSON object | CI/脚本读取最终结果、session id、usage |
| `streaming-json` | xAI/ACP 派生的 type-tagged NDJSON | 实时 UI、进度和工具事件 |
| `streaming-messages-json` | Messages API 风格 NDJSON | 需要重建 assistant/tool result message 的消费者 |

### `json` 的字段不都是必然存在

`text`、`stopReason`、`sessionId`、`requestId`、`num_turns` 和 usage/cost 取决于请求是否到模型、provider 是否返回完整 usage、子代理是否能汇总。缺少 `total_cost_usd` 不代表免费；可能是 server 没有提供完整 cost。

`total_tokens` 可以包含 cache buckets，而 `input_tokens` 在 headless spend projection 中是 uncached input；ACP PromptUsage 的 full prompt 语义又不同。账单脚本不能把不同协议里的同名字段直接相加。

### `streaming-json` 的事件顺序

典型事件是：

```text
thought
tool_call
tool_call_update
text
usage
end
```

还可能有 `plan`、`available_commands`、`max_turns_reached`、`auto_compact_*` 和 `error`。消费者应该按 `type` 做 forward-compatible switch，`end` 是当前 stream 的终点，不要把“收到一段 text”当成命令成功。

### `streaming-messages-json` 的 identity 陷阱

`system/init`、`assistant`、`user tool_result`、`result` 的消息结构更接近 Messages API。每条 emitted line 的 `uuid` 是该行的事件 id，不是 provider `message.id`，不能用它跨行去重；assistant message id 仍在消息内容里。`init` 里的 skills/tools/commands 是开始 stream 时的快照，运行中 reload 不会自动再发一个新的 init。

## 工具 allowlist、denylist 和 permission rule 是三层

| 机制 | 作用 |
| --- | --- |
| `--tools` | 只把指定 built-in tools 放进最终 toolset |
| `--disallowed-tools` | 从默认 toolset 移除工具，也可用 `Agent` 控制 subagent |
| `--allow/--deny` | 工具仍存在，但对具体调用做自动允许/拒绝/询问 |

`--disallowed-tools` 改变“模型看不看得到工具”，`--deny` 改变“看得到但能不能执行”。两者同时出现时，denylist 会覆盖 allowlist 选择。MCP meta-tools 还可能保持可见，需要按 user guide 和 config 一起确认。

## Session flags 的防覆盖语义

`-s/--session-id` 是创建新 session 的 UUID，不是旧版本的 upsert；恢复用 `-r/--resume`，继续最近 session 用 `-c/--continue`，`--fork-session` 才是从 resume/continue 复制到新 session。脚本应该从 JSON 取 `sessionId`，而不是根据 title 猜。

```bash
sid=$(grok -p "建立上下文" --output-format json | jq -r .sessionId)
grok -p "继续检查" --resume "$sid" --output-format json
```

## CI 的 stdout/stderr 设计

stdout 只放机器要解析的格式；日志写 stderr。`RUST_LOG` 在 headless 默认可以关闭，调试时再显式打开：

```bash
set -o pipefail
if ! result=$(RUST_LOG=error grok -p "review" --output-format json 2>grok.log); then
  cat grok.log >&2
  exit 1
fi
echo "$result" | jq -e '.text != null'
```

不要在命令替换中混入日志、warning 或 TUI ANSI。真实 CI 还要处理 auth、非零退出、`max_turns`、cancel、usage incomplete 和工作区 root。

## 本地验证

```bash
rg -n "OutputFormat|streaming-json|streaming-messages-json|include_partial|usage_is_incomplete|stopReason|headless" \
  crates/codegen/xai-grok-pager/src/headless \
  crates/codegen/xai-grok-pager/src/headless.rs
cargo test -p xai-grok-pager headless
```

优先运行 reducer tests，它们不需要真实模型，却能帮你理解每类输出如何从 ACP update 变成最终 JSON。
