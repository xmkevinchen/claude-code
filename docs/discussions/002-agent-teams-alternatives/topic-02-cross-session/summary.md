---
id: "02"
title: "跨 session 通讯与状态共享"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 跨 session 通讯与状态共享

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Agent Teams 的 mailbox 和 task list 基于文件系统，同一进程内的 agent 还有内存共享。没有 Agent Teams 后，多个 Claude Code 实例之间如何共享状态和通讯？

已知可用的跨 session 机制：
- 文件系统（任何 session 都能读写）
- Git（共享代码状态）
- `--resume` / `--continue`（session 链式执行）
- Durable cron（`.claude/scheduled_tasks.json`）
- Print mode + stream-json output

## Constraints
- 同上：不修改源代码，只用外部构建功能
- 多个 Claude Code session 可能同时运行在同一目录
- 需要处理并发写入冲突

## Key Questions
- 用文件系统模拟 mailbox 是否可行？延迟和可靠性如何？
- 多 session 并发写同一个文件的锁问题怎么解决？
- 有没有比文件更好的 IPC 方案（同一机器内）？
- session resume 的 context 开销是否可接受？
