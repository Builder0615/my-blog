# 30 TUI 主题、终端外观与配置

主题不是把几种颜色写进一个常量表。Grok Build 还要判断终端支持 truecolor/256/16 色，跟随系统深浅色，处理 Minimal 模式，选择语法高亮，设置 cursor color，并从 `pager.toml` 调整 scrollback 和 block。用户入口是 [`06-theming.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/06-theming.md)。

## 主题选择的输入

有效外观至少由这些输入决定：

```text
config theme / /theme
  + system appearance (auto)
  + terminal color capability
  + NO_COLOR
  + screen mode (fullscreen/inline/minimal)
  + pager.toml layout/block settings
  -> effective appearance
```

源码可以从 `xai-grok-pager-render/src/theme/mod.rs`、`theme/color_support.rs`、`theme/system_appearance.rs`、`theme/osc11.rs` 和 `appearance/cache.rs` 开始。

## 内建主题和 Minimal 的特殊性

当前用户指南列出 GrokNight、GrokDay、TokyoNight、RosePineMoon、OscuraMidnight，以及 `auto/system`。主题名大小写不敏感，可以通过 `/theme` 预览，也可以写入 `[ui].theme`。

Minimal 模式不是“全 TUI 主题变简洁”：它使用 terminal-native palette，忽略主题设置，`/theme` 和主题设置行不可用；语法高亮也不在 light/dark 文件之间切换。这个选择是为了直接写 native scrollback 和降低终端兼容问题。

## 颜色量化为什么会改变观感

主题内部可以用 RGB 描述，但实际终端可能只有 256 或 16 色。渲染层需要把颜色量化到可用 palette：

| 能力 | 结果 |
| --- | --- |
| truecolor | RGB 基本原样通过 |
| 256 color | 找最近的 indexed color |
| 16 color | 映射到 ANSI 名称 |
| `NO_COLOR` | 不输出颜色，使用 monochrome |

GrokNight/GrokDay 的中性灰量化后相对稳定；带色背景的主题在非 truecolor 终端可能失去原本的层次，因此 picker 可能隐藏它们。

## Auto theme 是一个 watcher 问题

macOS 读 `AppleInterfaceStyle`，Linux 查询 XDG Desktop Portal，Windows 读 personalization registry，SSH/headless 还可能用 OSC 11 背景查询。运行中每隔一段时间检测系统外观变化，再把 dark/light 映射到 `auto_dark_theme` 和 `auto_light_theme`。

要把“选中了 auto”和“系统改变后 UI 已经重绘”分开验证。前者是 config/state，后者还需要 watcher、theme cache、redraw request 和 terminal capability。

## `pager.toml` 不等于 `config.toml`

可以这样理解：

| 文件 | 主要控制 |
| --- | --- |
| `~/.grok/config.toml` | model、tools、auth、memory、permissions、features、session |
| `~/.grok/pager.toml` | terminal、animation、prompt、scrollback、block、todo 外观 |

`pager.toml` 的 `[scrollback.layout]`、`[scrollback.scrollbar]`、`[scrollback.scroll]`、`[scrollback.display]` 和 `[scrollback.blocks.*]` 共同决定一条消息怎样折叠、截断、显示 diff 和跟随尾部。

## 手动折叠和自动跟随是状态机

开启 `respect_manual_folds` 后，用户手工折叠的 block 不会被流式更新/finish 事件随意展开；用户展开时如果正在 follow tail，自动滚动会停下；`Shift+G`、滚到底部或发送新 prompt 才重新跟随。这个行为属于 scrollback state，不是 markdown renderer 的一个 bool。

## Cursor、语法高亮和退出恢复

主题切换会用 OSC 12 设置 cursor color，退出时用 OSC 112 恢复默认；代码 block 根据主题选择内建 `.tmTheme`；terminal restore 还要和 alternate screen、mouse、Kitty keyboard 一起处理。任何一步漏掉，用户看到的就是“颜色没恢复”或“退出后光标异常”。

## 本地实验

```bash
rg -n "color_support|quantiz|auto_dark_theme|AppleInterfaceStyle|OSC 12|theme::|respect_manual_folds" \
  crates/codegen/xai-grok-pager-render/src \
  crates/codegen/xai-grok-pager/src
```

先在不启动模型的情况下运行 theme/render 相关测试，再比较 truecolor、`TERM=xterm-256color`、`NO_COLOR=1` 和 `--minimal` 的输出。不要把截图差异直接归因于主题文件。
