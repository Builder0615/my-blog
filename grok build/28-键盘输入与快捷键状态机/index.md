# 28 键盘输入与快捷键状态机

快捷键不是一张静态速查表。一个 `Esc`、`Ctrl+C` 或 `Tab` 在不同焦点、模式、overlay 和 turn 状态下会做不同的事。用户指南是 [`03-keyboard-shortcuts.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/03-keyboard-shortcuts.md)，源码阅读入口是 `xai-grok-pager/src/input/key.rs`、`keyboard_normalizer.rs`、`app/agent_view/input.rs`、`app/dispatch` 和 `views`。

## 原始终端字节先变成统一事件

键盘输入不是直接拿到一个可靠的“按键字符串”：不同终端对 Kitty keyboard、CSI、Ctrl+字母、鼠标、paste 和 modifier 的编码不一样。输入层大致经过：

```text
terminal bytes
  -> crossterm / terminal parser
  -> keyboard normalizer
  -> Key / Action
  -> focus router
  -> modal / composer / scrollback / session dispatch
```

`keyboard_normalizer.rs` 的意义是把终端差异集中处理。若在 `AgentView` 里直接判断各种 ESC 序列，快捷键会随终端和 tmux 组合爆炸。

## Simple mode 和 Vim mode 改的是导航，不是所有输入

用户指南把输入状态分成两种：

| 状态 | 滚动区导航 | 进入 prompt |
| --- | --- | --- |
| Simple | Arrow、PageUp/Down、Shift+Arrow | `Tab`、`Space`，输入字母也可自动聚焦 |
| Vim | `j/k`、`h/l`、`g/G`、`H/L`、`e/E` | `i`、`Tab`、`Space` |

`[ui].vim_mode` 主要影响 scrollback navigation；prompt 编辑器的 multiline、shell mode、feedback mode 不是同一个模式。读代码时不要把 “vim” 传播成一个覆盖所有输入的全局判断。

## 焦点优先级比快捷键名称更重要

同一个 `Esc` 可能先被以下对象“偷走”：

1. modal、question card、permission view、slash/file completion；
2. history search、scrollback search、text selection、link highlight；
3. voice、shell mode、remember/feedback mode 的退出；
4. 当前 turn 的取消；
5. idle 时清空 prompt 或打开 rewind。

因此处理一键动作时，我会先画 `overlay -> focused pane -> turn state -> idle state` 的优先级。不要从快捷键表直接推断 `Esc` 永远取消。

## Esc 和 Ctrl+C 是两个不同的状态机

用户指南中特别容易被简化的一点是：

| 状态 | Esc | Ctrl+C |
| --- | --- | --- |
| turn running，普通/Minimal | 立即取消并保留 draft | draft 非空时先清 draft，空 draft 才取消 |
| turn running，fullscreen Vim | 被吞掉，不取消 | 用 Ctrl+C 或 command palette 取消 |
| cancelling | 再次发送取消，尝试补 ack | 可能升级到退出 |
| idle，有非空 prompt | 800ms 内双击清空 | 一次清空 |
| idle、空 prompt、有历史 | 800ms 内双击打开 rewind | 不承担 rewind |

cancel 后还有短暂的 rewind grace period，避免用户连续按 Esc 取消时误打开回滚选择器。这类细节不能只写成“Esc 取消，Ctrl+C 退出”。

## 问题卡片是一个独立焦点系统

`ask_user_question` 出现时，键盘由 question view 接管：上下键/j/k 选答案，左右键/h/l 切换问题，数字键直选，`z`/`Space` 进入自由文本，`Enter` 提交，`Esc` 取消当前选择，`Shift+X` dismiss。这里的 `Tab` 是在问题答案之间循环，不是普通的 prompt focus。

这说明 ACP 的“等待用户回答”会反向改变 TUI 输入路由。session 仍处于 turn 中，但 prompt composer 不能抢走所有按键。

## Agent-level 快捷键对应的系统动作

| 按键 | 作用 | 后续边界 |
| --- | --- | --- |
| `Ctrl+P` / `?` | command palette | slash/action registry |
| `Ctrl+M` | scrollback 时切模型，prompt focused 时切 multiline | model dispatch 或 composer |
| `Ctrl+O` | toggle always-approve | permission/session settings |
| `Ctrl+S` | session picker | session load |
| `Ctrl+B` | foreground command background 化 | task registry |
| `Ctrl+T` | todo pane | plan/todo state |
| `Ctrl+G` | tasks pane | subagent/background state |
| `F2` | settings modal | config setter/reload |

一个按键可能只改 UI，也可能发送 session command 或修改有效配置；要沿 `Action -> dispatch -> Effect -> ACP/session` 继续追。

## 外部 editor、paste 和媒体输入

Minimal mode 下外部编辑优先使用 `$VISUAL`，再使用 `$EDITOR`，随后 fallback 到 `vi`；保存后替换 draft，空文件清空 draft，但包含 file/image chips 的 prompt 不能被简单展平成纯文本。图片 paste 还要经过 terminal clipboard、压缩、媒体 block 和模型能力检查。

## 小实验

```bash
rg -n "esc_cancels_turn|Ctrl\+|KeyCode::Esc|vim_mode|question|keyboard_normalizer" \
  crates/codegen/xai-grok-pager/src/input \
  crates/codegen/xai-grok-pager/src/app \
  crates/codegen/xai-grok-pager/src/views
```

选 `Esc`，列出它在五个状态下命中的处理函数；再选 `Ctrl+M`，说明为什么同一个 key 在 prompt 和 scrollback 的语义不同。
