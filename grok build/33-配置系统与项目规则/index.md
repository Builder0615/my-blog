# 33 配置系统与项目规则

现有第 05 章讲了配置进入 Agent 的主链，这里把用户指南里的文件、字段和项目规则逐项展开。对应入口是 [`05-configuration.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/05-configuration.md) 与 [`12-project-rules.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/12-project-rules.md)。源码从 `xai-grok-shell/src/config/{mod,reloader,watcher}.rs`、`agent/folder_trust.rs`、`xai-grok-agent/src/discovery.rs`、`xai-grok-pager/src/config_toml_edit.rs` 开始。

## 三个配置文件先分家

| 文件 | 主要内容 | 影响范围 |
| --- | --- | --- |
| `~/.grok/config.toml` | model、auth、tools、MCP、memory、features、permissions、telemetry | 用户级/所有项目 |
| `~/.grok/pager.toml` | theme、layout、scrollback、animation、block rendering | TUI 外观 |
| `.grok/config.toml` | 项目 MCP、plugins、permission rules、tool-result cap 等允许的项目层配置 | 当前项目 |

还有 `.grok/skills`、`.grok/hooks`、`.grok/agents`、`.grok/lsp.json`、`.grok/sandbox.toml` 等目录/文件。不能把“项目配置”理解成把全局 `config.toml` 复制一份放进仓库；不同文件的允许 section 不同。

## 有效配置不是简单的文件覆盖

可以先用一张带来源的模型理解：

```text
built-in defaults
  -> user config
  -> managed/enterprise requirements
  -> environment variables
  -> project-scoped allowed config
  -> CLI / session overrides
  -> effective config
```

实际每个字段有自己的 merge 规则：模型默认可能按 model id 覆盖，MCP server 可能是项目同名完整替换，requirements 可能 clamp 某个值，CLI 可能只作用于本次 session。要以 `config/tests.rs` 的 fixture 和断言为准。

## 环境变量要按功能分类

用户指南里的变量可以分成：

| 类别 | 例子 | 读者容易犯的错 |
| --- | --- | --- |
| auth | `XAI_API_KEY`、`GROK_OIDC_*` | 以为设置了就一定赢过 session token |
| endpoint | `GROK_CLI_CHAT_PROXY_BASE_URL` | 把 proxy host 当成 model base URL |
| feature | `GROK_MEMORY`、`GROK_SUBAGENTS`、`GROK_WORKFLOWS` | 忽略代码中的默认值和 requirements |
| path | `GROK_HOME`、`GROK_RESPECT_GITIGNORE` | 改了目录却还读旧文件 |
| logging | `GROK_LOG_FILE`、`RUST_LOG` | 把 stderr 日志混进机器输出 |
| telemetry | `GROK_EXTERNAL_OTEL`、`GROK_TELEMETRY_*` | 把 coding-data privacy 与 OTEL 当成一开一关 |

## Project rules 的发现范围

Grok 识别 `Agents.md`、`Claude.md`、`CLAUDE.md`、`CLAUDE.local.md`、`AGENT.md`、`AGENTS.md` 等兼容文件，并扫描 `.grok/rules/`、受兼容开关控制的 `.claude/rules/`、`.cursor/rules/`。在 git repo 中会从 repo root 一直检查到当前 cwd；更深目录的规则出现在后面，冲突时优先级更高。

规则发现还受 gitignore、folder trust、compat settings 影响。自定义的 `AGENTS.local.md` 不一定会被顶层规则发现，但 rules directory 下的 `*.md` 又是另一套规则。

## 规则如何进入 prompt

```text
home rules
  + repo-root rules
  + nested directory rules
  + CLI --rules
  -> discovery result
  -> PromptContext / project-instructions message
  -> request builder
```

`--rules` 是追加到默认 system prompt 的 session 输入；`--system-prompt-override` 则替换默认 system prompt。项目规则文件通常整体加载，没有字符截断，因此每一行都会消耗上下文，应该写可执行约束而不是复制 README。

## LSP 配置也是来源合并

`~/.grok/lsp.json`、`.grok/lsp.json` 和 trusted plugin 的 `.lsp.json`/`lspServers` 都可以提供 server。优先级是 project > user > plugin；项目/用户同名条目替换低层，plugin 只能补没有被本地文件定义的名称。这个 merge 影响第 45 章的 LSP manager 和 prompt diagnostics。

## 热加载和信任

配置 watcher/reloader 让 theme、settings、skills、plugins、memory 等变化可以进入当前 runtime，但不是所有修改都能安全在 turn 中途生效。要分清：下一轮生效、立即改变 UI、需要重建 Agent、需要重启进程。project config、hooks、plugins 还会受 folder trust 约束。

## 本地检查

```bash
grok inspect
rg -n "load_effective_config|watcher|reloader|AGENTS|CLAUDE|lsp.json|trusted" \
  crates/codegen/xai-grok-shell/src/config \
  crates/codegen/xai-grok-shell/src/agent \
  crates/codegen/xai-grok-agent/src/discovery.rs
cargo test -p xai-grok-shell config
```

用临时目录构造“全局规则 + repo 规则 + 子目录规则”，观察 inspect 输出和 prompt context，不要把真实 home 配置拿来做覆盖实验。
