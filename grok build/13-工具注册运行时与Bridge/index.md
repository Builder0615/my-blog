# 13 工具注册、运行时与 Bridge

模型看到的工具是一份 schema，程序执行的工具是一段带资源和权限的 Rust 代码，中间需要一个桥。Grok Build 的工具系统把“声明工具”“找到实现”“解析调用”“执行并回传”分成了多层。

## `ToolRegistryBuilder` 做了什么

[`xai-grok-tools/src/registry/types.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-tools/src/registry/types.rs) 的 `ToolRegistryBuilder::new()` 会注册内建工具和多个 namespace。一个 registry entry 通常包含：

- namespace 和 tool id；
- `ToolKind`，例如 read、edit、execute、search、plan、web、background、memory；
- 需要的 capability 或 requirement；
- 给模型的 JSON schema；
- 参数验证和 dispatch closure；
- 工具实现以及结果转换。

仓库里可以看到 `GrokBuild`、`Codex`、`OpenCode`、MCP 和 hashline/concise 等命名空间。命名空间不是装饰：它让同名工具、兼容实现和不同参数协议可以共存。

## finalize 才是运行时组装点

registry builder 只知道工具目录。`finalize` 阶段才会注入 terminal、filesystem、cwd、session folder、notifications、skills、memory、web search、LSP、subagent state 等 resources，生成 `FinalizedToolset`。

这解释了为什么工具定义不能在程序启动最早期随便缓存：当前 workspace、权限规则、MCP server 和 session 能力都可能改变最终 toolset。

## 三个公共边界

| 边界 | 源码 | 解决什么 |
| --- | --- | --- |
| 工具实现 | `xai-grok-tools/src/implementations` | 读文件、bash、搜索、任务等具体动作 |
| runtime trait | [`xai-tool-runtime`](https://github.com/xai-org/grok-build/tree/ed6d543643628663873c5de28298e022ed634238/crates/common/xai-tool-runtime) | 统一 Tool、dispatch、error、notification、streaming |
| wire protocol | [`xai-tool-protocol`](https://github.com/xai-org/grok-build/tree/ed6d543643628663873c5de28298e022ed634238/crates/common/xai-tool-protocol) | JSON-RPC、注册、调用、progress、result 的消息类型 |

`xai-tool-runtime` 的 `Tool` trait 不等于模型 provider 的 tool schema；前者用于程序执行，后者用于模型选择调用方式。

## `ToolBridge` 为什么存在

[`xai-grok-tools/src/bridge.rs`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-tools/src/bridge.rs) 的 `ToolBridge` 持有 finalized toolset 和 terminal。它负责：

1. 给模型提供 definitions；
2. 把模型返回的名字和 JSON 参数解析成注册工具；
3. 处理 concatenated JSON 等输入恢复；
4. 执行或转交 MCP tool；
5. 把结果分成 UI/程序可读的 `ToolOutput` 和模型可读的 `prompt_text`。

**伪代码**如下：

```text
definitions = tool_registry.definitions()
call = model_response.tool_call
entry = registry.try_parse(call.name, call.arguments)
result = entry.dispatch(resources, validated_arguments)
return ToolBridgeResult(output_for_ui, prompt_text_for_model)
```

实际执行前还要经过 session 的 prepare、权限和 hook，不能把这段伪代码当成真实调用顺序的全部。

## 配置校验也是工具系统的一部分

registry 的 `validate_config` 会检查 preset/version、重复 client name、标准与 hashline file toolset 混用和 requirements。工具“注册成功”不等于“配置可用”；启动时可能先失败在配置校验，运行时再失败在权限或资源缺失。

## 从工具名字走到真实执行

一个工具至少要穿过这几层：

```text
tool definition
  -> namespace / kind / schema
  -> registry entry
  -> finalized toolset
  -> session-specific resources
  -> permission / hook / plan gate
  -> runtime dispatch
  -> model result + UI event + record
```

`ToolRegistryBuilder` 负责收集候选，`finalize` 负责把当前配置和环境确定下来，`ToolBridge` 让实现拿到 terminal、filesystem、cwd、session folder、memory、web 或 LSP 等资源。这个过程的意义是把“模型能看见的 schema”和“当前 session 真的能执行的实现”放在一起校验。

## schema 不是实现，也不是权限

schema 只告诉模型参数叫什么、类型是什么、描述如何写；实现还要解析参数、访问资源、处理超时和生成结果；权限则决定动作是否可以开始。三者分别对应：

| 层 | 典型错误 | 失败给谁看 |
| --- | --- | --- |
| schema/parse | 缺字段、类型不对、JSON 非法 | tool dispatcher / 模型 |
| implementation | 文件不存在、命令失败、网络错误 | tool result / UI |
| permission/policy | 被拒绝、需要确认、超出路径 | 用户、模型、session |

把它们分开，才能解释为什么“模型正确调用了工具”仍可能执行失败。

## finalized 的价值在于冻结上下文

同一工具定义在不同 workspace、不同 cwd、不同权限模式或不同 session folder 下，实际行为可能不同。finalized toolset 把这些运行时依赖绑定起来，避免工具执行到一半才发现没有 terminal 或 workspace resource。

动态 reload 时要注意旧 toolset 的生命周期：正在运行的调用是否继续使用旧资源？新 turn 是否看新 schema？plugin 被卸载后旧工具结果怎样归档？这类问题应该用 tool registry 和 session tests 一起回答。

## ToolBridge 是资源边界

bridge 的重点不是“把函数转发一下”，而是把多个资源组合成一个受约束的执行环境。它可能把 cwd、filesystem client、permission manager、session context、output limit、background task registry 传给实现。资源越多，越需要明确谁拥有它们、是否可 clone、是否跨线程、关闭时怎样清理。

读一个具体工具时，我会把参数、资源、AccessKind、错误转换和结果分成五列。缺一列，后面就很难判断它是普通读操作，还是会影响 workspace 的高风险动作。

## 小实验

```bash
rg -n "ToolRegistryBuilder|ToolNamespace|ToolKind|finalize|FinalizedToolset|try_parse|ToolBridgeResult" \
  crates/codegen/xai-grok-tools
```

任选 `read_file` 和 `bash`，做一张表：模型 schema、实现目录、资源依赖、AccessKind、输出上限和结果类型。不要只记工具名字。
