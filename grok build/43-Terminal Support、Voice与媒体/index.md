# 43 Terminal Support、Voice 与媒体

终端不是一个永远可靠的字符输入输出设备。不同 terminal emulator 对颜色、粘贴、modifier、鼠标、OSC、tmux、Zellij 和全屏模式的支持不一样；媒体输入也不是把路径丢给模型这么简单。用户入口是 [21-terminal-support.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/21-terminal-support.md)。

源码入口：

- xai-grok-pager/src/input/keyboard_normalizer.rs、line_editor.rs、terminal_support.rs、mouse.rs；
- xai-grok-pager-render/src/terminal、appearance、theme、clipboard；
- xai-grok-pager/src/app/dispatch/voice.rs、app/agent_view/paste.rs；
- xai-grok-tools/src/implementations/read_file/image.rs、metadata.rs、pdf.rs、pptx.rs；
- xai-grok-pager-render/src/clipboard/mod.rs、terminal/probe.rs、terminal/keyboard.rs 等平台适配。

## 终端能力探测为什么比“判断系统类型”可靠

同一台 macOS 机器上可能同时运行 Apple Terminal、iTerm、VS Code integrated terminal、tmux 或 Zellij。系统类型只能告诉我们操作系统，不能告诉我们 Enter modifier 是否被丢掉、是否支持 truecolor、是否有控制终端。

源码摘录：[terminal_support.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/src/input/terminal_support.rs) 的判断先检查 KeyCode，再结合终端能力：

~~~rust
pub fn is_mod_enter(key: &KeyEvent) -> bool {
    key.code == KeyCode::Enter
        && (key.modifiers.intersects(KeyModifiers::ALT | KeyModifiers::SHIFT)
            || is_apple_terminal_newline_modifier_held())
}
~~~

这段代码解决的是一个很具体的约束：Shift+Enter/Alt+Enter 应该产生换行，但 Shift+Tab 不应误判；某些 Apple terminal 丢掉 modifier 时，再用 OS probe 补偿。好处是把品牌差异集中在 capabilities；代价是探测、平台 API 和测试更复杂。

## 输入事件到文本的流程

~~~mermaid
flowchart LR
    K["terminal event"] --> N["keyboard normalizer"]
    N --> C{"当前 focus"}
    C -->|composer| L["line editor"]
    C -->|modal/permission| M["modal handler"]
    C -->|dashboard| D["dashboard dispatch"]
    L --> P["paste / @ reference / slash parser"]
    P --> T["prompt or command"]
    T --> A["ACP session prompt"]
~~~

图依据 input、app/dispatch、agent_view 和 ACP handler 的分层。它能说明事件路由，不代表每个按键都走同一个 parser；Esc、Ctrl-C、鼠标、voice submit 和外部 editor 都有特殊状态。

## 颜色、主题、OSC 和 screen mode

Terminal appearance 需要同时考虑：

| 层 | 内容 |
| --- | --- |
| theme | GrokNight、GrokDay、TokyoNight、RosePine、Oscura 等颜色 token |
| color support | truecolor、256、16 色量化 |
| terminal protocol | OSC 12 cursor、clipboard、mouse、alternate screen |
| screen mode | fullscreen、inline、minimal、headless |
| layout | pager.toml 中的 scrollback、blocks、prompt、terminal |

降级很重要。用户的 terminal 不支持 truecolor 时，主题仍要产生可读的 256/16 色；minimal 模式也不应因为缺少 dashboard overlay 就把 prompt 路由搞坏。主题是展示层，不应决定 session 是否完成。

## 粘贴、图片、PDF、PPTX 的边界

粘贴可能包含多行、控制字符、图片引用或大文本。pager 负责捕获和展示，真正的文件读取/媒体抽取在 tools 层。image、pdf、pptx reader 需要处理：

- 文件存在性、大小和 MIME/扩展名；
- 二进制、不可解析、加密或损坏；
- 抽取后文本/图片如何放进 content block；
- token/上传大小限制；
- permission、sandbox 和 artifact upload；
- 远端 client 不提供本地 filesystem 时的替代路径。

伪代码（媒体输入的边界）：

~~~text
attachment = parse_user_reference()
metadata = stat_and_classify(attachment)
if metadata.too_large or metadata.denied:
    return typed_error
content = reader_for(metadata.kind).extract(attachment)
block = make_acp_content_block(content)
send_to_session(block)
~~~

“模型支持图片”不意味着每个 provider、headless 输出格式或 ACP client 都支持图片；需要分别检查 sampler backend、content block、上传服务和客户端能力。

## Voice 为什么属于交互状态

voice 不只是一次 HTTP 请求。UI 还要知道 recording、interim transcript、submit、cancel、权限、设备错误和模型 turn 的对应关系。dashboard 和 status line 可能显示 voice 可用性，dispatch 还需要在 submit 时合并 interim 文本、停止录音，避免重复提交。

一个合理的状态图：

~~~text
idle -> recording -> interim transcript
  |          |              |
  |          +-- cancel ----+
  +-------------------------+
interim -> submit -> prompt
recording -> device error -> idle
~~~

## 失败路径

| 现象 | 可能层 |
| --- | --- |
| Enter 没有换行 | terminal capability、modifier rescue、tmux 转发 |
| Shift+Tab 被识别为换行 | KeyCode 判断顺序或 normalizer 回归 |
| 主题颜色刺眼/不可读 | color quantization、terminal 色阶、OSC 支持 |
| 粘贴大段文本卡住 | bracketed paste、truncation、prompt queue |
| PDF/PPTX 读取为空 | parser、权限、加密文件或 provider content 限制 |
| voice 文字重复 | interim 和 final transcript 合并逻辑 |
| headless 没有媒体 | stdout protocol 只支持文本/JSON，client 能力不足 |

## 本地验证

~~~bash
rg -n "is_mod_enter|keyboard_capabilities|terminal_context|paste|voice|read_file|image|pdf|pptx|OSC|truecolor|256" \
  crates/codegen/xai-grok-pager/src \
  crates/codegen/xai-grok-tools/src/implementations/read_file

cargo test -p xai-grok-pager terminal
cargo test -p xai-grok-pager keyboard
cargo test -p xai-grok-tools read_file
~~~

终端问题最好用最小复现：记录 terminal 类型、tmux/Zellij 状态、screen mode、按键序列和收到的 KeyEvent；只描述“回车坏了”不足以定位是哪层丢了信息。
