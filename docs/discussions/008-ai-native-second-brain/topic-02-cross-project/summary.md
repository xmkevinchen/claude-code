---
id: "02"
title: "跨项目记忆：是否属于 MVP Phase 1"
status: pending
current_round: 1
created: 2026-04-05
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 跨项目记忆

## Current Status
待团队讨论

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
AE pipeline 是 per-project 的——每个项目有自己的 discussions/plans/reviews。Claude Code 的记忆也是 per-project 的。但很多知识是跨项目有价值的："这个 API 超时要设 30s"在项目 A 学到，在项目 B 也有用。

OpenClaw 的 user-memory.prose 已经做了跨项目持久化（sqlite+ 后端，跨所有项目）。Claude Code 的 auto memory 也有全局级（~/.claude/memory/）。

## Constraints
- 跨项目需要解决命名空间问题（项目 A 和项目 B 的同名文件/模块可能完全不同）
- 隐私：项目 A 的信息是否应该在项目 B 中被搜到？（个人级别可能 OK，团队级别需要考虑）
- 存储：per-project 用文件就行，跨项目需要数据库

## Key Questions
- 跨项目知识的实际价值有多大？什么类型的知识是跨项目有价值的，什么是 project-specific？
- 如果 Phase 1 不做跨项目，架构上要预留什么？
- 如果 Phase 1 做跨项目，最小实现是什么？
