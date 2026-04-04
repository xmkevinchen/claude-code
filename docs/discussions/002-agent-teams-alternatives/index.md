---
id: "002"
title: "Agent Teams 替代方案"
status: active
created: 2026-04-04
pipeline:
  analyze: skipped
  discuss: done
  plan: pending
  work: pending
plan: ""
tags: [agent-teams, alternatives, orchestration]
---

# Agent Teams 替代方案

如果 Agent Teams feature 最终没有 ship，作为 Claude Code 用户，在不修改源代码的情况下如何实现多 agent 协作。

## Problem Statement

Agent Teams 提供了：团队创建/销毁、文件 mailbox 通讯、共享任务列表、idle/wake 生命周期、worktree 隔离。如果这些被移除，需要找到不依赖这些内部机制的替代方案。

## Topics

| # | Topic | File | Status | Decision |
|---|-------|------|--------|----------|
| 1 | 多 agent 协作的替代架构 | [topic-01-multi-agent/](topic-01-multi-agent/) | converged | 三层方案：AgentTool background / 文件 IPC / 外部编排器 |
| 2 | 跨 session 通讯与状态共享 | [topic-02-cross-session/](topic-02-cross-session/) | converged | 命名输出文件，不依赖 transcript resume |
| 3 | 长时间任务编排与恢复 | [topic-03-orchestration/](topic-03-orchestration/) | converged | 外部编排器 + claude -p + worktree + SQLite |

## Documents
- [Conclusion](conclusion.md)
