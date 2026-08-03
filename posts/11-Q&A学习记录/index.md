# Q&A 学习记录

本文件记录学习过程中“没看懂、发起二次提问”的问题。

## 记录格式

每条记录固定包含：

```markdown
### Qn：<一句话问题>

- 日期：YYYY-MM-DD
- 背景：<为什么会有这个问题>
- 源码位置：<可选的链接>
- 回答摘要：<关键答案>
- 是否解决：是 / 否
```

## 已记录问题

### Q1：Pi 和 Grok Build 应该以哪个为学习蓝本？

- 日期：2026-08-03
- 背景：开始学习 Agent 开发，面临两个候选仓库，不确定先读哪个。
- 源码位置：
  - [Pi](https://github.com/earendil-works/pi)
  - [Grok Build](https://github.com/xai-org/grok-build)
- 回答摘要：先以 Pi 为蓝本。理由：Pi 是 TypeScript 分层的 agent toolkit，核心循环在单个可读文件里，有架构文档、SDK 示例和评测；Grok Build 是 Rust 大仓库（约百万行、近百 crate），适合掌握基础后研究生产级细节，不适合入门通读。
- 是否解决：是

## 待记录区

以下区域保留给后续提问：

```markdown
### Q3：<等你来写>
```
