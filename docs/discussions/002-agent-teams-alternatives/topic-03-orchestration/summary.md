---
id: "03"
title: "长时间任务编排与恢复"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 长时间任务编排与恢复

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Agent Teams 下，leader 可以创建任务列表、分配给 teammate、teammate 自动认领和执行。没有 Agent Teams 后，需要一个外部编排方案来管理多个 Claude Code 实例的任务分配、执行、失败恢复。

已知可用的编排机制：
- Durable cron（定时触发）
- Print mode + bash 脚本（外部调度）
- tmux session 保活
- Git worktree（隔离工作目录）
- GitHub Actions（云端事件驱动）
- Session resume（断点续跑）

## Constraints
- 需要支持任务失败后的自动恢复
- 需要支持并行任务执行
- 需要不阻塞用户的正常工作
- 需要可观测性（知道什么在跑、什么完成了、什么失败了）

## Key Questions
- 外部 orchestrator 脚本的最小可行架构是什么？
- 如何实现任务状态追踪而不依赖 Claude Code 内部的 TaskList？
- 多个 print mode 实例并行跑同一个 repo 会有什么冲突？
- worktree 隔离 + print mode 的组合能否完全替代 Agent Teams + worktree？
