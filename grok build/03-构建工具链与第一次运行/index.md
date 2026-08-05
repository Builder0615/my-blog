# 03 构建工具链与第一次运行

我建议把“构建失败”和“源码看不懂”分开处理。Grok Build 的构建闭包包含 pinned Rust toolchain、DotSlash、`protoc` 和平台相关条件；如果这些前置没准备好，错误信息可能完全还没进入 Agent 代码。

## README 明确要求的依赖

当前 README 的构建说明包含：

- Rust 由 [`rust-toolchain.toml`](https://github.com/xai-org/grok-build/blob/ed6d543643628663873c5de28298e022ed634238/rust-toolchain.toml) 固定。
- DotSlash 负责运行 `bin/` 下的 hermetic 工具，特别是 `bin/protoc`。
- `protoc` 可以由 DotSlash 提供，也可能回退到 PATH 或 `$PROTOC`。
- macOS 和 Linux 是支持的构建主机，Windows 从这个公开树构建属于 best-effort。

先检查环境：

```bash
rustc --version
cargo --version
dotslash --help
protoc --version
```

`dotslash` 不在 PATH 时，不要直接开始分析 Rust 报错。先按仓库 README 的安装说明处理工具链，再重跑最小命令。

## 从轻到重的命令

```bash
# 只读 workspace 元数据，不编译全部源码
cargo metadata --no-deps --format-version 1

# 检查单个 leaf 或边界 crate
cargo check -p xai-grok-sampling-types
cargo check -p xai-grok-pager-bin

# 运行仓库 README 给出的单 crate 测试
cargo test -p xai-grok-config

# 格式与 lint
cargo fmt --all
cargo clippy -p xai-grok-tools
```

官方 README 特意建议 `cargo check -p <crate>`，因为 full-workspace build 很慢。教程也采用同样的验证策略：先让一个边界 crate 通过，再进入上层。

## 第一次运行不必立即发模型请求

```bash
cargo run -p xai-grok-pager-bin -- --help
cargo run -p xai-grok-pager-bin -- agent --help
```

这两个命令能验证 binary、clap 参数和基础启动路径，不需要先完成一轮模型调用。真正运行 TUI 或登录时，才会涉及认证、浏览器、终端接管和外部网络。

## package 名、binary 名和产品命令名

这里有三个名字：

| 名字 | 位置 | 含义 |
| --- | --- | --- |
| `xai-grok-pager-bin` | Cargo package | 组合根 package |
| `xai-grok-pager` | release artifact | Rust binary artifact |
| `grok` | 官方安装命令 | 产品分发名 |

一个常见误会是看到 `pager` 就以为这是普通分页器。README 明确说这个 binary 包含 TUI 和 Agent runtime 的产品入口。

## 把错误归类

| 错误位置 | 先问什么 |
| --- | --- |
| toolchain / rustup | 是否使用了仓库 pinned toolchain |
| DotSlash / protoc | hermetic 工具能否下载和启动 |
| Cargo resolution | 是否在正确的 workspace 根目录 |
| 单 crate 编译 | 当前 crate 的 feature 或依赖是否满足 |
| 启动时认证 | 是否需要浏览器、API key 或 device flow |
| TUI 行为 | terminal 类型、tmux、PTY、screen mode 是否匹配 |

这样分层后，编译失败不会被误写成“Grok Build 架构如此”，终端异常也不会被误写成“Agent loop 有 bug”。

## 构建阶段和运行阶段要分开

我会把第一次运行拆成四个阶段：

| 阶段 | 发生什么 | 常见误判 |
| --- | --- | --- |
| 工具准备 | rustup、DotSlash、`protoc` 和平台工具就位 | 以为 Cargo 能自动解决所有工具 |
| 依赖解析 | workspace package、外部 crate、feature 被解析 | 把网络下载失败当成源码错误 |
| 编译链接 | build script、生成代码、library 和 binary 产物形成 | 只盯着末尾一条 error，忽略最早的根因 |
| 启动运行 | 参数、配置、认证、terminal、session 被创建 | 把登录失败当成编译失败 |

`cargo check` 主要帮助我确认类型和依赖，`cargo build` 才会完整生成可执行产物；测试还会引入 dev-dependency 和 test feature。一个 crate 能 `check` 不代表 release profile、特定平台或真实终端一定能运行。

## 第一次运行不必一上来连接真实服务

可以按风险从低到高推进：

```bash
cargo metadata --no-deps --format-version 1 > /tmp/grok-metadata.json
cargo check -p xai-grok-sampling-types
cargo check -p xai-grok-pager-bin
cargo run -p xai-grok-pager-bin -- --help
```

`--help` 是很好的第一条产品路径：它能验证 binary、参数解析和一部分启动依赖，但不必消耗模型额度或写入会话。真正需要登录时，再单独记录认证需求；不要把凭据写进 shell history、源码或 Markdown。

## 看到编译错误时先定位“拥有者”

Rust 错误常常从一个 leaf crate 传到上层 binary。比如一个类型定义改动，可能让 sampler、session、TUI 同时报错。我的处理顺序是：

1. 找最早出现的 error，而不是 warning 数量最多的文件。
2. 看报错类型属于 workspace crate、外部 crate、build script 还是平台条件。
3. 用 `cargo check -p <owner>` 缩小范围，再回到产品 binary。
4. 只有在源代码和 feature 都确认后，才判断是不是上游 API 变化。

如果错误只在 `#[cfg(unix)]`、`#[cfg(feature = "...")]` 或测试模块出现，笔记里必须记住这个条件。否则读者复制命令时可能在另一平台得到完全不同的结果。

## 小实验

执行 `cargo metadata` 后，用 `jq` 找到八个主路径 package，并把 manifest path 贴回第一章的源码地图。再用 `cargo check -p xai-grok-sampling-types` 对照：一个只描述消息的 crate，和 binary package 的编译成本有什么不同？
