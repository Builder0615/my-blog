# 17 Workspace、文件系统、VCS 与 Worktree

Agent 的“工作区”不只是 cwd 字符串。它包含文件系统、命令执行、Git/Jujutsu 状态、checkpoint、worktree 和可能的远程 workspace exposure。把这些能力集中在 workspace crate，是为了让工具不直接到处调用 `std::fs` 和 `Command`。

## workspace crate 的边界

[`xai-grok-workspace/src/lib.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-workspace/src/lib.rs) 导出 `WorkspaceOps`、`WorkspaceSession`、`WorkspaceClient`、`WorkspaceEvent`、权限和工具配置类型。可以先把它分成三层：

1. **能力**：读写文件、执行命令、读取 VCS、创建 checkpoint。
2. **会话**：当前 cwd、workspace identity、事件和权限上下文。
3. **客户端/远程**：当文件系统或 terminal 由 ACP client 提供时，调用路径如何变化。

`xai-grok-workspace-types` 和 `xai-grok-workspace-client` 说明内部实现和外部调用可以分开演进。

## VCS 不是展示信息

Git status、diff、repo root 和 branch 会进入 TUI、提示词、worktree、rewind 或分享流程。对应的支撑 crate 包括 `xai-gix-status`、`xai-fast-worktree` 和 shell 的 `session/worktree*`、`extensions/git.rs`。

读 VCS 相关代码时，区分：

- 为模型提供的状态摘要；
- 为用户显示的 diff/status；
- 为隔离任务创建的 worktree；
- 为 rewind/checkpoint 回滚使用的记录。

它们可能都叫“Git 信息”，但权限和失败方式不同。

## worktree 解决什么问题

当用户希望把 Agent 工作放到独立分支或目录时，worktree 能让修改与当前工作树隔离。它带来额外生命周期：创建、命名、关联 session、处理已有目录、结束后保留或清理、进程中断时恢复。

不要把 `--worktree` 解释成“复制一份项目”。它更接近 VCS 提供的另一个工作目录，具体 branch/ref 行为要看 `session/worktree.rs`、`worktree_pool.rs` 和 `xai-fast-worktree`。

## checkpoint 和 rewind

checkpoint 记录一个能被恢复或比较的工作状态；rewind 是 session/UI 让用户回到更早状态的能力。它可能同时涉及 conversation leaf、工具结果、文件变更和 VCS 操作，不能只看 history。

建议分开追两条链：

```text
session rewind -> history/leaf/ChatState
workspace rewind -> files/VCS/checkpoint
```

然后找它们在哪个 command 或 extension 里汇合。

## WorkspaceOps 为什么值得抽出来

如果每个工具都自己处理 cwd、路径归一化、远程调用、权限和 VCS，读写行为会在多个地方悄悄分叉。`WorkspaceOps` 这类边界让工具先表达“我要读/写/执行/查状态”，再由当前 workspace session 决定如何实现。

这层抽象还方便测试：同一个 file tool 可以接本地临时目录、fake client 或远程 workspace client，而不需要在每个工具里重新模拟 Agent runtime。代价是初学者需要多跳几次定义；读到 trait 时要顺着具体 impl 和构造函数找回来。

## cwd、repo root 和 workspace identity

三个路径不要混为一谈：

| 名字 | 白话解释 | 为什么重要 |
| --- | --- | --- |
| cwd | 当前命令/工具默认工作的目录 | 相对路径和 shell 执行位置 |
| repo root | VCS 识别出的仓库根 | status、diff、worktree、规则发现 |
| workspace identity | 当前 session 绑定的工作区标识 | leader 复用、远程能力和持久化 |

cwd 可能是仓库子目录，repo root 可能不存在，远程 workspace 甚至没有本地 path。提示词里告诉模型的目录和 permission manager 用来匹配的目录也必须对齐，否则会出现“模型能读但不能写”或“规则没有命中”。

## rewind 的两个风险

文件 rewind 可能改变用户刚刚手工修改的内容，历史 rewind 可能改变模型下一次能看到的分支。两者如果由一个 UI 操作同时触发，需要明确顺序和失败补偿：文件回滚成功但 history 回滚失败怎么办？history 回滚成功但 Git 操作失败怎么办？

checkpoint 不一定是完整文件快照，也可能记录 diff、VCS ref 或可重做操作。学习时要确认它的 storage format 和恢复权限，不要看到“checkpoint”就承诺可以无条件恢复到任何状态。

## worktree 的清理比创建更难

创建 worktree 需要处理已有路径、branch 冲突和正在运行的 child process；清理时要确认 session 是否仍引用它、未提交改动是否需要保留、VCS 删除是否成功，以及 crash 后下次启动能不能发现孤儿 worktree。成功路径只占一半，真正值得读的是中断和重复运行测试。

## 远程 workspace exposure

leader protocol 的 `workspace_exposure` capability 和 ACP client 的 `fs_read`/`fs_write` 宣告，说明某些模式下 Agent 的文件能力可能由客户端提供。不要看到 `xai-grok-workspace-client` 就断言所有文件操作都远程化；要根据 client capabilities 和 session/new metadata 判断。

## 小实验

```bash
rg -n "WorkspaceOps|WorkspaceSession|checkpoint|rewind|worktree|repo_root|git" \
  crates/codegen/xai-grok-workspace \
  crates/codegen/xai-grok-shell/src/session \
  crates/codegen/xai-grok-shell/src/extensions
```

画一张“普通 cwd、独立 worktree、远程 client filesystem”的对照表，列出文件写入者、VCS 状态来源和失败后的恢复方式。
