# 24 学习路线与动手实验

我不会要求你把 81 个 package 从上到下读完。更有效的方式是每次闭合一条链，链上的每个节点都能回到源码、测试或命令验证。下面的实验从不触网开始，逐渐增加状态和外部依赖。

## 阶段 A：建立语言和地图

1. 读 00，能解释 crate、trait、async、channel、actor、turn、session。
2. 跑 `cargo metadata --no-deps`，把 81 个 package 按第一章分组。
3. 对照 `main.rs`、`cli.rs` 和 `lib.rs` 画出入口图。

产物是一张“package → 运行时职责 → 代表文件”表，而不是一段空泛文字。

## 阶段 B：只读请求链

不启动真实模型，沿 `SessionCommand::Prompt → build_request → SamplerHandle → SamplingEvent` 读类型。重点记录：

- request 在哪一步加入 session/turn metadata；
- stream event 如何变成 response；
- error、cancel 和 usage 分别由谁处理。

可以用：

```bash
rg -n "Prompt|build_request|submit_and_collect|SamplingEvent|record_response_token_usage" \
  crates/codegen/xai-grok-shell/src/session \
  crates/codegen/xai-chat-state \
  crates/codegen/xai-grok-sampler
```

## 阶段 C：闭合一个 read 工具链

选 `read_file`，完成：schema → registry → parse → permission → workspace read → truncate → tool result → next request。不要一次选 MCP 或 subagent；先让同步 read 工具建立感觉。

## 阶段 D：闭合一个 edit/bash 链

选 `search_replace` 或 `bash`，加上 hook、permission、path lock、子进程、sandbox、post-tool result。回答“用户取消发生在写入前、写入中、写入后，各是什么结果”。

## 阶段 E：长任务和恢复

用源码和测试比较：

- 普通 turn 与 max turns；
- auto compaction 与手动 compact；
- memory flush 与 session persistence；
- leader reconnect 与 auto-update relaunch；
- subagent 完成与 parent session update。

这一步会把第 11、19、20、21、23 章连起来。

## 阶段 F：才进入 TUI

选择一个用户动作，例如 `Esc` 取消、`Ctrl+O` 切换 always-approve、`Tab` 切换 focus 或 `/compact`：

```text
keyboard event -> dispatch -> session/permission command
                -> ACP/session event -> AppView state
                -> Presenter -> writer thread -> terminal
```

如果能完整追完一个动作，就已经理解 TUI 的主结构，不需要先读所有主题和 glyph。

## 给完全没读过 Rust 的读者

可以把每阶段的产物固定下来，遇到不懂的语法也不会迷路：

| 阶段 | 你应该留下的东西 |
| --- | --- |
| 地图 | package、入口、状态拥有者、源码路径 |
| 请求链 | 一条带类型名的时序图 |
| 工具链 | 一个工具从 schema 到 result 的表 |
| 长任务 | 取消、恢复、压缩和持久化的状态图 |
| TUI | 一个按键到 terminal bytes 的路径 |
| 同步审计 | snapshot、测试证据、未确认问题 |

如果读到 trait、宏或 lifetime 卡住，可以先把它当作“实现某个接口的约束”，继续找 impl 和调用方。只有当它影响线程、生命周期、错误或 feature 行为时，再回头补 Rust 细节。

## 一个最小的脱离产品练习

在理解 00、02、09、10、13 后，可以自己写一个只支持：一个模型接口、`read_file` 和 `echo` 两个工具、一个 `Vec<Message>`、一次 tool loop 的小程序。它不需要 TUI、持久化或真实 sandbox，但要自己处理：错误 result、最大轮数、工具参数解析和取消。

写这个小程序的目的不是替代 Grok Build，而是验证你是否真的理解：模型为什么会再次调用、tool result 怎样回到上下文、什么时候退出循环。能解释它和生产实现少了哪些边界，就说明对照阅读有效。

## 每次实验都写预期和反例

不要只写命令。每个实验至少补三行：

```text
预期看到：哪个 package / 类型 / event
如果没有看到：最可能缺哪个 feature、权限或输入
这次不能证明：哪些平台、真实模型或并发场景仍未覆盖
```

这会把“命令跑过了”变成可复查的学习记录，也方便源码更新后重新执行。

## 做一次小型上游同步演练

保存当前 `HEAD`、`SOURCE_REV`、package 数量、用户指南文件名和主入口路径。假设下一次 `cargo metadata` 少了一个 package 或 `main.rs` 函数移动，先标出索引、源码链接、正文和 Q&A 中需要检查的地方，再更新文章。演练的重点是更新方法，而不是预测上游会怎样改。

## 和 Pi 对照着学习

Pi 适合先建立“model API → agent loop → AgentSession → TUI/SDK”的第一版模型。Grok Build 再把这版模型拆开，补上多 actor、permission、sandbox、leader、MCP、memory 和生产级恢复。对比的单位应该是问题和调用链，不是类名。

前一版文档的问题正是跳过了这条渐进过程：拿 Pi 的 10 个主题当目录，就把 Grok Build 的前置知识和产品扩展压掉了。现在章节多不是为了显得完整，而是让每次阅读的跨度小一点。

## 每次同步源码后的检查单

```bash
git rev-parse HEAD
cat SOURCE_REV
cargo metadata --no-deps --format-version 1
rg -n "pub (struct|enum|trait)|fn main|run_loop|turn.rs" crates
```

然后检查索引、源码链接、调用图和 Q&A 是否仍然指向同一个 snapshot。能证实的内容更新，不能证实的内容标为待查，不用靠推测填空。
