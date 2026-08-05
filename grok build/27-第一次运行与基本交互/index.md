# 27 第一次运行与基本交互

前面的 03 讲的是“怎样把代码构建出来”，这里讲的是普通用户第一次执行 `grok` 后到底发生什么。对应的用户入口是 [`01-getting-started.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/01-getting-started.md)，源码入口主要看 [`xai-grok-pager-bin/src/main.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager-bin/src/main.rs)、[`xai-grok-pager/src/app/session_startup.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/src/app/session_startup.rs) 和 `xai-grok-pager/src/app/agent_view`。

## 安装成功不等于 Agent 已经能工作

第一次使用可以拆成四个独立检查：

| 检查 | 证明了什么 | 还没有证明什么 |
| --- | --- | --- |
| `grok --version` | binary 在 PATH 上，版本能打印 | 认证、模型、终端、工具 |
| `grok --help` | CLI 参数解析和早退路径可用 | TUI 和 session |
| 浏览器/API key 登录 | credential 能拿到并保存 | token 刷新、模型请求 |
| 发一个只读 prompt | session、sampler、TUI 基本闭环 | 编辑、shell、sandbox、恢复 |

这张表能把“安装问题”“登录问题”和“Agent 行为问题”分开。用户指南中的安装脚本、PowerShell 路径和版本更新属于产品分发；源码学习则从 binary 入口继续追参数和 startup intent。

## TUI 里看到的两个主要区域

交互界面可以先抽象成两个 view：

```text
scrollback
  - 用户 prompt
  - assistant markdown / thinking
  - tool call / diff / terminal output
  - todo、session、system notice

prompt composer
  - 普通文本
  - @ 文件引用
  - / slash command
  - ! shell mode / # feedback 等模式
```

这不是两个孤立的文本框。`AgentView` 保存当前输入、焦点、选中 block、turn 状态和 overlay；`AppView` 再处理全局 modal、多个 agent/session 和路由。消息进入 ACP handler 后，UI 才把它分成 scrollback block，而不是让 session 直接渲染终端。

## `@` 文件引用为什么值得单独理解

`@src/main.rs`、`@src/main.rs:10-50` 和 `@src/` 是给模型附加上下文的入口。用户看到的是 fuzzy picker，代码需要处理：

- 当前 cwd 和 workspace root；
- 文件/目录区分；
- 行范围解析；
- `.gitignore` 和隐藏文件过滤；
- `!` 前缀对隐藏文件搜索的改变；
- 选择结果怎样变成 prompt content block 或 file attachment。

相关实现可以从 `xai-grok-pager/src/views/file_search`、`app/agent_view/input.rs`、`app/agent_view/paste.rs` 和 `xai-grok-tools/src/implementations/read_file` 开始。`@` 只是 UI 入口，真正的文件读取仍要经过工具和权限边界。

## 第一轮 prompt 的状态变化

```text
输入文本
  -> composer 解析 @、/、! 等模式
  -> dispatch 识别普通 prompt
  -> session startup / ACP session/prompt
  -> ChatState 构建 request
  -> sampler stream
  -> ACP update
  -> AgentView 把消息变成 scrollback blocks
```

如果输入的是 `/model` 或 `/compact`，它可能在 composer/command registry 里被消费，不进入普通模型 prompt；如果输入的是 `!ls`，它会走 shell mode；如果输入普通文字，才进入第 12 章的 turn 主链。

## 常用启动参数背后的语义

| 用户写法 | 对应的产品意图 |
| --- | --- |
| `grok "fix the test"` | 启动 TUI 并立刻提交第一轮 |
| `grok --cwd <path>` | 改变 workspace/cwd 起点 |
| `grok --worktree=<name> "..."` | 在独立 worktree 中开始 session |
| `grok --rules "..."` | 为本次 session 添加规则 |
| `grok --yolo` | 修改 permission mode，不是 sandbox |
| `grok -m <model>` | 选择本次使用的模型 |
| `grok --resume <id>` | 加载既有 session |
| `grok -c` | 当前目录继续最近 session |
| `grok --minimal` | 选择 scrollback-native screen mode |
| `grok -p "..."` | 进入 headless，不创建 TUI 交互 |

参数只表达意图，后面仍会经过 config、auth、leader/sandbox policy 和 session setup。不要把 `--yolo`、`--minimal` 或 `--worktree` 当成一个简单的全局 bool。

## 第一次排错从证据开始

建议按这个顺序收集信息：

```bash
grok --version
grok --help
grok doctor --json
RUST_LOG=debug grok -p "只读取当前目录并列出三个文件" 2>/tmp/grok-headless.log
```

不要把真实 credential 放进日志。`stdout`、`stderr`、退出码、当前 cwd、模型、sandbox profile 和 session id 应该分开记录。这样下一步才能判断是 binary、auth、request、tool 还是 TUI 的问题。

## 本地源码实验

```bash
rg -n "Command::|AgentCmd|HeadlessArgs|PagerArgs|session_startup|file_search" \
  crates/codegen/xai-grok-pager-bin/src \
  crates/codegen/xai-grok-pager/src/app \
  crates/codegen/xai-grok-pager/src/views
```

沿着一个普通 prompt 找到 `dispatch`、session startup 和 ACP handler 后，再用第 28 章跟踪一个按键。两个入口之后会汇合到 session，但它们在进入 session 前的状态处理完全不同。
