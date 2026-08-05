# 29 Slash Commands 与命令分发

Slash command 是 TUI 里最容易被误认为“特殊 prompt”的功能。它有自己的 registry、matcher、参数解析、模式限制、autocomplete、别名和 dispatch。用户入口是 [`04-slash-commands.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/04-slash-commands.md)，源码从 `xai-grok-pager/src/slash/registry.rs`、`slash/command.rs`、`slash/matcher.rs`、`slash/mode_support.rs` 和 `app/dispatch` 开始。

## `/` 输入经过哪条路径

```text
composer sees '/'
  -> slash matcher finds command candidates
  -> dropdown shows name / description / argument hint / source
  -> Enter or Tab accepts
  -> command parser validates arguments
  -> slash command handler returns action/effect
  -> UI change, session command, config write, or external process
```

普通模型 prompt 不需要知道 `/compact` 是什么；command handler 可以直接提交 compact、new session、model switch 或打开 modal。只有某些命令会把文本转成 follow-up prompt。

## 命令不是一个平面列表

用户指南的命令可以按副作用分组：

| 组 | 代表命令 | 主要结果 |
| --- | --- | --- |
| Session | `/new`、`/resume`、`/fork`、`/rewind`、`/export`、`/delete` | session picker、history、文件恢复 |
| Model/Mode | `/model`、`/effort`、`/plan`、`/always-approve`、`/auto`、`/minimal` | 修改当前 Agent/session/UI 状态 |
| Context | `/compact`、`/context`、`/remember`、`/flush`、`/dream` | 压缩、memory 或 context 查询 |
| Extension | `/hooks`、`/plugins`、`/marketplace`、`/skills`、`/mcps` | 打开管理界面或 reload |
| Task/Workflow | `/loop`、`/goal`、`/workflow`、`/workflows`、`/deep-research` | 创建后台运行或状态化流程 |
| Media | `/imagine`、`/imagine-video` | 发起媒体生成请求 |
| UI/诊断 | `/theme`、`/settings`、`/timestamps`、`/doctor`、`/dashboard` | 配置、外观、诊断或 session 管理 |
| Account | `/login`、`/logout`、`/usage`、`/privacy`、`/feedback` | 认证、计费、数据设置、反馈 |

这些命令的副作用不同，不能都写成“调用一个函数”。例如 `/rewind` 会影响文件和 history，`/theme` 只影响 UI/config，`/goal` 可能启动 workflow driver。

## 别名和冲突解析

`/t` 可以是 `/theme`，`/m` 可以是 `/model`，`/undo` 可以是 `/rewind`。启用的 skill 还可能以 `user-invocable: true` 出现在 slash menu 中。内建命令优先于同名 skill，但可以用 scope-qualified name，例如 `/local:commit`。

这意味着 matcher 要同时保存名称、别名、来源 scope、参数说明和优先级。一个命令从 dropdown 消失，可能是 skill disabled、scope 不可信、插件关闭或 mode_support 不允许，不一定是 parser 失效。

## 参数解析和错误路径

命令参数至少要处理：无参数时打开 picker、参数数量、quoted text、路径/ID、未知选项、命令进行中是否可用。失败时应该留在 composer 或打开错误提示，而不是把错误文本发给模型。

例如 `/resume` 可以按 session id 或 title 搜索，`/theme tokyonight` 是直接选择，`/theme` 无参数则打开 picker/循环，`/loop 5m <prompt>` 还需要解析时间间隔和剩余文本。命令定义的 parser 和 handler 需要一起读。

## Slash command 也会遇到生命周期问题

命令可能发生在 turn running、permission pending、question card、minimal mode 或 reconnect 期间。`/compact` 需要排队到安全点，`/login` 可能触发 auth refresh，`/plugins` 可能更新工具和 prompt，`/quit` 要让 session flush/terminal restore 有机会完成。mode_support 测试是理解这些限制的好入口。

## 本地实验

```bash
rg -n "SlashCommand|SlashCommandRegistry|mode_support|alias|Autocomplete|user-invocable" \
  crates/codegen/xai-grok-pager/src/slash \
  crates/codegen/xai-grok-pager/src/views/slash_dropdown.rs \
  crates/codegen/xai-grok-pager/src/app
```

选 `/compact`、`/model`、`/rewind` 各一条：画出输入文本怎样变成 handler，标注它是 UI action、session command、配置写入还是模型 prompt。
