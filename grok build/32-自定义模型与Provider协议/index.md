# 32 自定义模型与 Provider 协议

“换模型”并不是替换一个字符串。Grok Build 需要解析模型目录、选择 API backend、解析凭据、计算 context window、生成 headers/query params，再把统一的 request 交给 sampler。用户入口是 [`11-custom-models.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/11-custom-models.md)，源码从 `xai-grok-shell/src/agent/models/{resolution,endpoint,fetch,cache}.rs`、`xai-grok-models/src/lib.rs` 和 `xai-grok-sampler/src/client.rs` 读起。

## 模型解析的输入和产物

模型选择可能来自 CLI `-m`、`/model`、`Ctrl+M` picker、`[models].default`、内建模型目录、远端 `/v1/models` 或 `[model.<id>]` 自定义配置。解析产物至少要包含：

```text
model slug / provider id
display name + description
api backend
base URL + auth source
context window
sampling defaults
extra headers / query params
capabilities and feature gates
```

模型 picker 能显示，不等于 endpoint 一定可访问；endpoint 能访问，也不等于当前模型支持所有 tool/image/reasoning 能力。

## 三种 API backend

用户指南当前列出：

| `api_backend` | 请求协议 | 适合追的 sampler 文件 |
| --- | --- | --- |
| `chat_completions` | OpenAI Chat Completions | `stream/chat_completions.rs` |
| `responses` | OpenAI Responses | `stream/responses.rs` |
| `messages` | Anthropic Messages | `stream/messages.rs` |

统一层需要把各 provider 的 text delta、thinking、tool use、usage、stop reason 和 error 变成共同事件；但某些 provider 特有字段不能假装完全等价。读 adapter 时要同时看序列化 request 和反序列化 stream。

## 自定义模型配置的字段意义

```toml
[model.my-model]
model = "provider-model-id"
base_url = "https://api.example.com/v1"
name = "My Model"
api_backend = "chat_completions"
context_window = 128000
temperature = 0.7
top_p = 0.95
max_completion_tokens = 8192
env_key = ["PROVIDER_KEY", "FALLBACK_KEY"]
extra_headers = { "X-Tenant" = "team-a" }
query_params = { "api-version" = "2026-07-22" }
```

`context_window` 不只是 UI 显示，它会影响 compaction。新模型未写这个字段时可能采用默认值，若和 provider 实际窗口不一致，会出现请求过大或过早压缩。

## 凭据和 headers 的优先级

模型级 `api_key` 高于 `env_key`，后者可以是多个环境变量名并取第一个非空值；没有模型级 credential 时再考虑 session token 和全局 `XAI_API_KEY`。`extra_headers` 还分 global `[models]` 和 per-model，per-model 按 key 覆盖 global 同名项，其他 global header 保留。

headers 可以用于 provider auth、租户、成本归因；但模型配置是敏感边界，不能把 key 写入共享仓库或日志。请求 tracing 需要做 redaction。

## 全局默认值与单模型覆盖

temperature、top_p、max completion、reasoning effort、headers 等可能有全局默认和模型级覆盖。读 config merge 时要区分“字段未设置”“设置成空值”“设置成 0”三种情况，不能用 `unwrap_or` 把它们混掉。

同一个模型从远端目录得到的默认值，也可能被本地 `[model.<id>]` 覆盖。用户指南的 priority order 要和 `agent/models/resolution.rs` 的实际合并顺序对照。

## Web search、summary、image 是不同模型消费者

默认模型、web search 模型、session summary 模型、image description 模型和 subagent model 可以各自解析。不要看到 `/model` 切换成功，就推断 compaction 或 web tool 也换成了同一个模型；调用方会把不同用途的 slug 传给不同 sampler request。

## 如何判断 connection error 属于哪一层

| 现象 | 先查 |
| --- | --- |
| model not found | catalog/slug/resolution |
| 401 | credential precedence/header |
| 404 | base_url、API backend、path 拼接 |
| 400 schema | provider adapter/request body |
| stream parse error | `stream/*` event decoder |
| context overflow | context window、token estimate、compaction |
| tool 不可用 | model capability/agent toolset，不一定是 endpoint |

## 本地实验

```bash
rg -n "api_backend|context_window|extra_headers|env_key|query_params|model_overrides|resolve.*model" \
  crates/codegen/xai-grok-shell/src/agent/models \
  crates/codegen/xai-grok-shell/src/config \
  crates/codegen/xai-grok-sampler/src
cargo test -p xai-grok-sampler --lib
```

可以写一个不发网络的配置测试，断言 `api_key > env_key > session/global` 和 global/per-model header merge；再用 sampling server 测试不同 backend 的 stream 事件。
