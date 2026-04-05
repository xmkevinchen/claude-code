---
id: "01"
title: "隐藏系统与未完成功能"
status: pending
current_round: 1
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 隐藏系统与未完成功能

## Current Status
待团队调查

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Claude Code 源码中有多个 feature-flagged 或半完成的子系统：autoDream（后台记忆整合）、Buddy（宠物养成）、KAIROS（主动助手）、ULTRAPLAN（远程 Opus 规划）、UDS_INBOX（Unix Domain Socket 消息）、hooks 系统（87 个文件）、plugins 系统、OAuth 多提供商等。这些系统的完成度、设计意图和扩展潜力尚未被系统性梳理。

## Constraints
- 这是源码镜像，无法运行验证
- Feature flags 控制的代码可能在 bundle 时被 DCE（Dead Code Elimination）剔除
- 部分代码可能是内部实验性功能，不面向公众

## Key Questions
- 哪些子系统功能完整但被 flag 关闭？哪些是半成品？
- 有没有"隐藏的宝藏"——设计精巧但不为人知的系统？
- 哪些子系统暗示了 Anthropic 的产品路线图方向？
