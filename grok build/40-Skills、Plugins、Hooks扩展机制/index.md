# 40 Skills、Plugins、Hooks 扩展机制

这三种扩展都能改变 Agent 的能力，却不在同一个层面：

| 机制 | 解决什么 | 进入运行时的时机 |
| --- | --- | --- |
| Skill | 一份可按需注入模型上下文的工作方法 | 发现、预加载或模型调用时 |
| Plugin | 打包 agent、skills、hooks 等文件的分发单元 | 安装、信任、发现、reload |
| Hook | 在 session、tool、compact、stop 等事件旁执行外部逻辑 | 事件触发时 |

用户入口是 [08-skills.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/08-skills.md)、[09-plugins.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/09-plugins.md) 和 [10-hooks.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/10-hooks.md)。

源码入口：

- xai-grok-agent/src/discovery.rs：agent、skill、plugin scope 和优先级发现；
- xai-grok-tools/src/implementations/skills/skill.rs：skill 输入、输出与 skill envelope；
- xai-grok-hooks/src/event.rs、config.rs、discovery.rs、dispatcher.rs、result.rs：事件、匹配、执行、决策；
- xai-grok-plugin-marketplace/src/catalog.rs、index.rs、installer.rs、scanner.rs、types.rs：浏览、安装、路径安全、扫描；
- xai-grok-shell/src/plugin.rs：主应用的安装、trust、remote pin 和 reload 接线。

## Skill 是“上下文注入”，不是魔法函数

源码中的 Skill 输出包含 skill_name、可选 skill_message 和错误信息；formatter 会把内容包装成 skill envelope。它的好处是模型可以按需获得长篇工作方法，主 system prompt 不必永远塞满；代价是 skill 内容会消耗 context，并且不可信 skill 可能改变模型行为。

伪代码（压缩 skill tool 到 prompt 的路径）：

~~~text
name, args = parse_skill_request()
info = registry.resolve(name, scope, cwd)
if info.missing:
    return typed_skill_error
body = read_skill_file(info.path)
message = build_skill_message(info, body)
return tool_result(message)
~~~

不要把 slash command、skill tool 和 plugin file 当成三份重复功能：slash 是用户入口，skill tool 是模型可调用入口，plugin 是供应和安装这些定义的容器。

## Hook 事件表为什么集中生成

源码摘录：[event.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-hooks/src/event.rs) 用一张宏表同时生成事件枚举、别名、显示名和 dispatch traits：

~~~rust
hook_events! {
    SessionStart {
        display: "session_start",
        aliases: ["SessionStart", "session_start", "sessionStart"],
        traits: (Observe, Tested, true),
    },
    PreToolUse {
        display: "pre_tool_use",
        aliases: ["PreToolUse", "pre_tool_use", "preToolUse"],
        traits: (Tool, Tested, false),
    },
}
~~~

这样新增事件时不容易只改了 parser 却忘了 display、matcher 或 hub forwarding。宏的代价是初学者不容易从单一 enum 定义看懂完整行为，需要顺着生成表和 traits 读取。

## Hook 的阻塞边界

pre_tool_use 会按配置顺序执行匹配的 hooks；明确的 deny 才阻止 tool。快照还把 timeout、崩溃、找不到命令和 malformed output 归为 Failed，按 fail-open 继续 tool，并把失败展示或记录。这里不能写成“hooks 一定保护住工具”：这是当前实现的安全假设，受其运行环境和源码策略限制。

| Hook 结果 | 对 tool/turn 的影响 |
| --- | --- |
| Allow | 继续 |
| Deny | 阻止当前 tool，带 reason |
| Failed | 当前 dispatcher 按 fail-open 继续，记录失败 |
| Stop block | 影响 turn 是否继续 |
| additionalContext | 将额外信息送回 session/模型路径 |

## Plugin 安装为什么要做路径和信任分层

Marketplace 文件名来自外部输入，MarketplaceRelativePath 会拒绝绝对路径、..、当前目录组件、平台前缀和越过根目录的路径；这是防止解压或安装路径逃逸的基本边界。远程 plugin 还涉及 ref、SHA、组织策略、trust 和 plugin scope。未信任 plugin 的 agent 文件可以只解析 frontmatter，不直接注入 body，这是降低“安装即执行或注入”的方式之一。

安装完成后的流程可以画成：

~~~mermaid
flowchart TD
    U["用户安装 plugin"] --> C["catalog/index resolve"]
    C --> P["校验来源、ref、SHA、相对路径"]
    P --> S{"是否可信"}
    S -->|否| F["frontmatter-only / 禁用 body"]
    S -->|是| I["install files"]
    I --> D["discover agents + skills + hooks"]
    F --> D
    D --> R["registry/reload"]
    R --> H["session 使用扩展"]
    H --> K["hook dispatch / skill expansion / agent spawn"]
~~~

图的依据是 marketplace types.rs 的路径错误类型、installer/scanner、xai-grok-agent/discovery.rs 的 plugin agent 加载分支和 hooks dispatcher。它不能证明所有 plugin 都无副作用；外部命令、HTTP hook 和 skill body 仍需按不可信输入对待。

## 三者叠加时的优先级

一个请求可能同时受这些层影响：

1. plugin 是否安装、启用、可信；
2. scope 是 project、user、bundled 还是 built-in；
3. discovery 对同名定义怎样去重；
4. skill 是否被预加载或由模型调用；
5. hook 是否匹配当前 event；
6. permission/sandbox 是否允许真正的 tool。

所以“我装了 plugin 但命令没有出现”可能是 discovery、scope、trust、reload 或 registry visibility 问题，不一定是安装失败。

## 失败路径

| 现象 | 排查 |
| --- | --- |
| skill 找不到 | 名称限定、scope、frontmatter、reload |
| plugin 安装成功但 agent 不可见 | trust 导致 body 不加载、文件名或目录不合规 |
| hook 未触发 | event alias、matcher、enabled、trust-disabled |
| hook 超时仍执行 tool | 这是当前 fail-open 策略，要看 HookRunResult::Failed |
| 远程 plugin 被拒 | unpinned ref、SHA policy、组织限制 |
| 同名 agent 选择错误 | discovery.rs 的 scope priority 和 project override |

## 本地验证

~~~bash
rg -n "build_skill_message|HookEventName|PreToolUse|HookDecision|fail-open|MarketplaceRelativePath|frontmatter|trusted|remote_sha|scope" \
  crates/codegen/xai-grok-tools/src/implementations/skills \
  crates/codegen/xai-grok-hooks/src \
  crates/codegen/xai-grok-plugin-marketplace/src \
  crates/codegen/xai-grok-agent/src/discovery.rs \
  crates/codegen/xai-grok-shell/src/plugin.rs

cargo test -p xai-grok-hooks
cargo test -p xai-grok-plugin-marketplace
cargo test -p xai-grok-agent discovery
~~~

## 外部资料怎么用

社区文章适合提供安装体验和常见配置，但 plugin、hook、skill 的安全判断必须回到源码和测试。外部材料应该标明作者、链接和访问日期，不能用“别人这样写”替代当前 commit 的行为证据。

