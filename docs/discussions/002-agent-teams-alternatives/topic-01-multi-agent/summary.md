---
id: "01"
title: "多 agent 协作的替代架构"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 多 agent 协作的替代架构

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Agent Teams 提供了 TeamCreate/SendMessage/TaskCreate 等工具实现 in-process 多 agent 协作。如果被移除，需要找到只用 Claude Code 公开功能（AgentTool、print mode、cron、hooks）实现多 agent 并行工作的方案。

AgentTool 本身是无条件发布的（不需要 Agent Teams feature flag），`run_in_background` 和 `isolation: "worktree"` 也是独立功能。问题是没有 SendMessage/TeamCreate/TaskList 时如何做 agent 间的协调。

## Constraints
- 不修改源代码
- 只用外部构建中可用的功能
- AgentTool 可用（无 feature gate）
- run_in_background 可用
- isolation: "worktree" 可用
- CronCreate 可用（GA，默认开启）
- print mode (-p) + session resume 可用
- Hooks 系统可用
- 文件系统可读写

## Key Questions
- 没有 SendMessage 时，agent 之间如何通讯？
- 没有 TaskList 时，如何做任务分配和进度跟踪？
- 没有 idle/wake 机制时，如何让 agent 持续运行并响应新任务？
- 现有的 `<task-notification>` 机制（background agent 完成后的通知）是否足够替代 mailbox？
