# 31 认证全流程与凭据刷新

认证不是一个“登录成功/失败”的布尔值。Grok Build 当前同时支持浏览器 OAuth、API key、OIDC/企业 SSO、device code 和 external auth provider；凭据还要保存、刷新、热加载，并参与模型/endpoint 的选择。用户入口是 [`02-authentication.md`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/crates/codegen/xai-grok-pager/docs/user-guide/02-authentication.md)，源码从 `xai-grok-shell/src/auth` 的 `credential_provider.rs`、`manager.rs`、`external_auth.rs`、`oidc` 和 `refresh` 开始。

## 认证链里有四个角色

| 角色 | 回答的问题 |
| --- | --- |
| credential source | token/API key 从哪里来 |
| credential storage | 以什么文件/内存格式保存 |
| auth manager | 什么时候复用、刷新、失效、重试 |
| request client | 怎样把 credential 放进 endpoint 请求 |

认证 manager 不应该直接决定工具权限。登录成功只说明模型服务的请求可能有 credential；workspace、permission、sandbox 决定 Agent 能不能读写主机。

## 浏览器登录和 API key 的优先级

用户指南给出的请求级优先级是：

```text
per-model api_key / env_key
  > active session token in ~/.grok/auth.json
  > XAI_API_KEY fallback
```

交互登录默认写入 `~/.grok/auth.json`，MCP OAuth token 另有 `~/.grok/mcp_credentials.json`。Unix 下文件默认 owner-only；这不是“文件加密”，有权访问用户目录的进程仍可能读取 token。

`grok logout` 清掉缓存后，API key 才会在没有 session token 时成为 fallback。读代码时要确认“登录页面显示的账号”“request 实际选中的 key”“模型级自定义 key”是不是同一条链。

## OIDC 是 Authorization Code + PKCE

企业 SSO 需要 issuer、client id、loopback redirect、scope 和可选 audience。CLI 通过 issuer 的 `/.well-known/openid-configuration` 发现授权端点，浏览器完成登录，回调后保存 token；PKCE 让公共 CLI client 不需要 client secret。

这条路径的失败点包括 issuer discovery、redirect 端口、scope/audience、IdP 回调、refresh token 和 proxy endpoint。不能把“浏览器能打开”当成 OIDC 成功。

## External Auth Provider 最重要的是 stdout/stderr 合约

当用户配置 `auth_provider_command` 时，Grok 会用 `sh -c` 启动它：

```text
external binary stdout -> token / token JSON -> Grok 保存
external binary stderr -> URL、进度、错误 -> Grok 展示给用户
exit 0                 -> credential accepted
non-zero / empty       -> provider failed, fallback or re-auth
```

stdout 不能混入日志。token 可以是裸字符串，也可以是包含 `access_token`、`refresh_token`、`expires_in`、`issuer` 的 JSON。想让 Grok 知道 token 何时过期，就提供 `expires_in` 或配置 `auth_token_ttl`。

## `GROK_AUTH_EXPIRED` 改变运行合约

外部 provider 被调用时，`GROK_AUTH_EXPIRED=1` 表示 headless refresh：stdin 关闭、stderr 不一定有人看、运行时间很短、不能等待用户交互。变量未设置则更像交互式登录，允许浏览器/device-code 过程。

这个细节解决了一个很实际的问题：一个需要用户输入的 provider 如果在后台刷新时永远等待，会拖慢每次启动。provider 应该在 expired contract 下快速成功或快速失败，让 Grok 回到 sign-in 路径。

## 自动刷新和 401 重试

凭据可能在三种时机刷新：

1. 过期前的 proactive refresh，由 `expires_in`/TTL 和 `GROK_AUTH_EARLY_INVALIDATION_SECS` 决定；
2. 服务返回 401 后的 refresh + request retry；
3. 外部修改 `auth.json` 后，下一次 API request 的 hot reload。

一次 auth retry 还要考虑并发：多个 sampler request 不能同时启动多次浏览器/刷新，应由 single-flight/lock 让一个请求更新 credential，其他请求等待或复用结果。

## 认证失败的时间线

| 失败点 | 可能结果 |
| --- | --- |
| 没有 credential | TUI 显示登录，headless 返回错误/尝试 provider |
| token 解析失败 | 不应把半截 token 发给 API |
| refresh 失败 | 标记旧 credential 无效，重新登录 |
| 401 | 刷新后有界重试，避免无限循环 |
| provider 超时 | kill 子进程并展示 stderr/诊断 |
| auth 成功但模型不可用 | 继续看 endpoint/model resolution，不要归咎登录 |

## 本地验证

```bash
rg -n "auth_provider_command|GROK_AUTH_EXPIRED|expires_in|early_invalidation|single_flight|refresh|auth.json" \
  crates/codegen/xai-grok-shell/src/auth \
  crates/codegen/xai-grok-shell/src/config
cargo test -p xai-grok-shell auth --no-default-features
```

不要用真实 token 做实验。可以用临时目录和假的 provider 输出，单独验证 stdout JSON 解析、non-zero、空输出和 TTL 分支。
