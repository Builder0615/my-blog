# 14 文件、Shell 与搜索工具

这是最适合小白做第一次端到端源码跟读的一组工具：读文件、列目录、搜索、替换和执行命令。它们看起来简单，却正好穿过模型 schema、工具解析、权限、workspace、终端和结果截断多个边界。

## 源码目录和用户名字可能不同

实现主要在 [`xai-grok-tools/src/implementations`](https://github.com/xai-org/grok-build/tree/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-tools/src/implementations)，Grok Build 自己的实现目录包括：

| 用户能理解的能力 | 代码目录/文件 | 主要问题 |
| --- | --- | --- |
| 读文件 | `grok_build/read_file` | 路径、行范围、编码、输出大小 |
| 列目录 | `grok_build/list_dir` | cwd、隐藏项、结果排序 |
| 搜索 | `grok_build/grep` | ripgrep 参数、匹配输出和限制 |
| 编辑 | `grok_build/search_replace` | 替换是否唯一、写入是否可验证 |
| shell | `grok_build/bash` | command、PTY、退出码、输出和中断 |
| 兼容实现 | `codex`、`opencode`、hashline 目录 | 不同工具协议和编辑策略 |

用户指南里可能把 bash 能力描述成 `run_terminal_command`，而源码 namespace/id 可能使用 `bash` 或其他内部命名。判断行为时以 registry entry 和实现 dispatch 为准。

## read 工具的安全边界

read_file 不只是 `fs::read_to_string`：它需要把模型给出的相对路径绑定到 workspace/cwd，处理行范围和二进制/超大文件，限制输出，并把错误变成模型可理解的 tool result。权限系统还可能把 read 归类为 `AccessKind::Read`。

读实现时，沿这条链走：

```text
tool schema -> arguments -> path normalization -> workspace fs -> truncation -> ToolOutput
```

每个箭头都要问：路径是否改变？错误是否带敏感绝对路径？输出是展示给用户还是发给模型？

## edit 工具不能只看“写文件”

`search_replace` 要检查旧文本是否匹配、匹配次数是否符合预期、写入前后内容、换行和编码。并发调用同一个写路径时，session tool executor 还会使用路径锁。兼容的 Codex/OpenCode/hashline 实现则说明编辑协议可以有多种输入形式。

写工具最终还要经过：plan mode edit gate、pre-tool hook、permission decision、workspace 写入、post-tool hook 和结果回传。第 16、17 章分别讲这些外层边界。

## bash 工具的关键对象

执行命令时不要只记 command string。还要找到：

- command 是直接 child process 还是 PTY；
- stdout/stderr 是否流式记录；
- exit code、signal 和 timeout 如何变成结果；
- 用户 interjection 或 cancel 怎样影响进程；
- sandbox 是否限制子进程 filesystem/network；
- 长输出在哪一层截断，截断提示如何告诉模型。

对应的 terminal 适配在 `xai-grok-shell/src/terminal`，本地 terminal、PTY、background task 和 ACP terminal 路径可能不同。

## 搜索工具是 Agent 的“眼睛”

grep/search 不只是便利功能。它让模型在不读取整个仓库的情况下定位定义，也影响 token 消耗和误判概率。搜索实现要同时处理 glob、隐藏文件、`.gitignore`、二进制文件、结果上限和 cwd。

## 文件工具为什么比“读写文件”复杂

`read_file` 至少要决定编码、行范围、超大文件、二进制内容、路径解析和输出截断；`edit` 还要决定匹配是否唯一、换行符是否保持、文件不存在时是否创建、写入失败时是否留下临时文件。写工具的安全性不只来自 permission，还来自实现本身的校验。

我会用三个例子理解实现边界：

| 输入 | 需要确认的行为 |
| --- | --- |
| 读取不存在的路径 | 返回结构化错误，还是空文本 |
| 替换文本出现两次 | 拒绝避免误改，还是全部替换 |
| 写入超出 workspace 的路径 | 路径归一化后由谁拒绝 |

模型能否正确恢复，取决于 error result 是否说明了下一步。一个只有“command failed”的错误会让模型盲目重试；一个包含路径、匹配数量、退出码和截断状态的结果，才适合继续推理。

## Shell 工具是进程生命周期管理

执行 shell 不只是调用 `Command::new`：要设置 cwd、环境变量、stdin/stdout/stderr、超时、取消、PTY 或 background task；要处理子进程退出、信号、输出回压和残留进程；还要把 sandbox、permission 和 workspace client 的限制传进去。

终端输出通常是两份：一份面向用户的实时 event，一份面向模型的截断后结果。不要假设两份内容完全相同，也不要把 ANSI 控制序列原样塞给模型。

## 搜索工具在控制上下文成本

搜索结果上限不是纯 UI 选择。结果过多会消耗 token，过少可能让模型漏掉定义；忽略隐藏文件或 `.gitignore` 可能让结果与用户本地经验不一致；二进制过滤和编码错误也要明确。一次搜索最好保留查询、cwd、glob、限制和是否截断，方便解释模型为何没找到某段代码。

## 错误要沿三条路径传播

一个文件或 shell 错误可能同时进入：工具 result（给模型）、tool event（给 UI/ACP）和 session record（给恢复/诊断）。三条路径格式不同、时机不同。追 `ToolOutput`、truncate、terminal event 和 record 函数时，我会把它们画成分叉，而不是假定有一个全局 error handler。

## 小实验

```bash
rg -n "struct .*Read|search_replace|run_terminal|ToolOutput|truncate|AccessKind" \
  crates/codegen/xai-grok-tools/src/implementations \
  crates/codegen/xai-grok-shell/src/terminal \
  crates/codegen/xai-grok-workspace
```

选一个错误案例，例如文件不存在、替换文本出现两次、命令退出码非零，写出“模型看到的错误”和“用户看到的错误”是否相同。
