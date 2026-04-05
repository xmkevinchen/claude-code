---
id: "01"
title: "MVP Phase 1 scope 和方案"
status: pending
current_round: 1
created: 2026-04-05
decision: ""
rationale: ""
reversibility: ""
---

# Topic: MVP Phase 1 scope 和方案

## Current Status
待团队讨论

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Product vision 定义了 MVP Phase 1 为"个人闭环"：Claude Code session hook + AE pipeline output watcher + 反哺 hook + Dreaming。需要具体化这个方案——技术选型、数据流、接口设计、最小功能集。

AE 是 per-project 的（每个项目有自己的 .ae/ 或 docs/ 目录）。Claude Code 的记忆也是 per-project 的（~/.claude/projects/{project}/memory/）。MVP Phase 1 需要决定：是 per-project 的 Second Brain，还是从一开始就跨项目？

## Constraints
- 底层尽可能复用 OpenClaw 组件（Context Engine、Dreaming、memory_search）
- 不侵入 Claude Code 或 AE plugin 核心代码——通过 hooks 接入
- 目标用户是工程师，CLI + Markdown 优先
- Phase 1 要能验证闭环是否成立

## Key Questions
- MVP Phase 1 的最小功能集是什么？哪些可以砍？
- 技术上怎么实现"AE output watcher"——文件 watcher、hook、还是手动触发？
- "反哺 hook"具体怎么注入 AI 工作流？往 system prompt 里加还是 user context？
- Phase 1 就做跨项目还是先 per-project？
