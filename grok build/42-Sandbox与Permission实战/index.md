# 42 Sandbox 与 Permission 实战

Permission 和 sandbox 经常被初学者当成同一件事。它们都能阻止危险操作，但发生在不同层：

| 层 | 回答的问题 | 典型实现 |
| --- | --- | --- |
| Permission | 这一次 tool call 是否需要用户同意 | rule matching、交互授权、always approve |
| Sandbox | 即使进程被允许运行，它能访问哪些资源 | macOS Seatbelt、Linux Landlock/bwrap、网络策略 |
| Tool policy | Agent 是否看得到某种 tool | capability、mode、allow/disallow |
| Hook | 外部策略是否追加 allow/deny 或 stop | pre_tool_use、stop gate |

用户入口是 [18-sandbox.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/18-sandbox.md) 和 [22-permissions-and-safety.md](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/22-permissions-and-safety.md)。源码主要看 xai-grok-sandbox/src、xai-grok-workspace/src/permission 和 pager 的 permissions dispatch。

## 一个操作如何被放行

伪代码（把多个 crate 的边界压缩在一起）：

~~~text
tool_call = parse_model_tool_call()
if not tool_registry.exposes(tool_call.kind):
    return tool_not_available

permission = resolve_user_policy(tool_call, cwd, mode)
if permission == Ask:
    decision = client.request_permission(tool_call)
    if decision != Allow:
        return permission_denied

profile = resolve_sandbox_profile(session, cwd)
process = spawn_under_profile(tool_call, profile)
return collect_tool_output(process)
~~~

这个顺序体现了几个设计选择：tool 不可见时不应该绕到 permission；用户同意也不应自动扩大 OS 能力；sandbox apply 失败不能静默当成成功。

## ProfileName 是安全配置的入口

源码摘录：[profiles.rs](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-sandbox/src/profiles.rs) 把内置 profile 和 custom profile 分开：

~~~rust
pub enum ProfileName {
    Workspace,
    Devbox,
    ReadOnly,
    Strict,
    Off,
    Custom(String),
}
~~~

Profile 最终会解析为 read_only、read_write、deny、write_deny、default_read 和 restrict_network 等能力。它不是一个单纯字符串开关，所以阅读配置时要跟到 to_capability_set、deny glob、child network 和平台 feature。

## Permission 和 Sandbox 的组合关系

~~~mermaid
flowchart TD
    A["模型提出 tool call"] --> B{"tool registry 是否暴露"}
    B -->|否| X["返回不可用"]
    B -->|是| C["permission rule matching"]
    C -->|deny| D["拒绝并记录原因"]
    C -->|ask| E["客户端请求用户决定"]
    C -->|allow| F["解析 sandbox profile"]
    E -->|拒绝| D
    E -->|允许| F
    F --> G{"平台/feature 可执行"}
    G -->|否| H["受控失败或降级"]
    G -->|是| I["spawn child under kernel policy"]
    I --> J["stdout/stderr/exit code/event log"]
~~~

图的依据是 xai-grok-workspace permission resolution、xai-grok-sandbox profiles/deny、tools shell runner 和 pager permission view。图能说明“用户许可”和“进程能力”是两道门，不能证明某个 profile 在每个操作系统都提供相同强度。

## 内置 profile 的取舍

| Profile | 适合 | 代价/风险 |
| --- | --- | --- |
| off | 调试兼容性、确认是否由 sandbox 导致失败 | 没有 OS 级限制 |
| workspace | 普通项目编辑 | 仍要检查 workspace 外的默认读取和子进程 |
| devbox | 需要更宽开发环境的任务 | 可访问面更大 |
| read-only | 代码阅读、分析 | 测试或工具可能需要写临时文件 |
| strict | 高约束、网络受限 | 容易遇到依赖、证书和终端兼容问题 |
| custom | 项目/组织的精确策略 | 配错会拒绝合法操作或暴露过多路径 |

用户指南中的 yolo/always approve 改变的是询问策略，不应被解释成“关闭 sandbox”。同样，read-only permission 也不一定等于 OS 层完全不可写：要看 tool 自己的临时目录、子进程环境和 profile 的 read_write。

## Custom profile 为什么要求 additive 合并

源码说明 global sandbox.toml 和 project .grok/sandbox.toml 的关系：项目配置可以增加新名字，但不能重定义全局已经存在的 custom profile。原因是恶意 workspace 不应该通过同名配置把企业/用户 profile 的 deny 清空、read_write 放宽。

这是一个容易被忽略的“配置来源信任”问题：

~~~text
global profile = trusted baseline
project profile = allowed extension
same-name overwrite = rejected
parse error = log + do not invent a permissive policy
~~~

如果改成简单的 last-write-wins，配置代码更短，却把 workspace 文件变成了安全策略的提权入口。

## 平台和进程边界

macOS 主要依赖 Seatbelt；Linux 根据 feature 使用 Landlock 或 bwrap 等机制。child process 网络限制和 in-process API 的网络能力并不是一回事：shell 子进程可能被 restrict_network 限制，Agent 自身已经建立的 HTTP client 不会因此神奇地断网。

Sandbox 还是进程级/子进程级的边界。恢复 session 时固定 profile 的选择，能避免用户在旧 session 中打开了 strict，后来配置改变却悄悄变成宽松模式；具体快照还要查看 session startup 和 profile resolution。

## 失败路径

| 现象 | 可能原因 |
| --- | --- |
| sandbox apply 失败 | 平台不支持、feature 没开、路径不存在、设备节点不可打开 |
| 合法命令被拒 | read-only、deny glob、cwd 解析或临时目录未加入 |
| 网络命令失败 | child network policy，而不是模型 API 本身 |
| yolo 仍不能执行 | policy pin、组织限制、sandbox 或 tool 不可用 |
| custom 配置没生效 | scope、同名冲突、TOML parse error |
| hook 说 allow 但仍被拒 | hook 只影响事件决策，OS profile 仍可拒绝 |

不要用“把 profile 改成 off”作为唯一修复；它只适合做诊断对照。找到具体路径、平台能力或权限规则后，再缩小地修改配置。

## 本地验证

~~~bash
rg -n "ProfileName|SandboxProfile|read_only|read_write|deny|restrict_network|CapabilitySet|Landlock|Seatbelt|permission" \
  crates/codegen/xai-grok-sandbox \
  crates/codegen/xai-grok-workspace/src/permission \
  crates/codegen/xai-grok-shell/src \
  crates/codegen/xai-grok-pager/src/app/dispatch

cargo test -p xai-grok-sandbox
cargo test -p xai-grok-workspace permission
~~~

做实验时记录操作系统、编译 feature、profile 名、tool 类型、cwd 和完整错误；同一条命令在 macOS、Linux、headless 和 TUI 下不一定得到同样结果。

