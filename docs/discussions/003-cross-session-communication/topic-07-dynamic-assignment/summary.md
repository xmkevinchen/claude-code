---
id: "07"
title: "动态任务分配与队员调度"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 动态任务分配与队员调度

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Agent Teams 的 TaskList + idle/wake 机制让 leader 可以动态分配任务——teammate 完成一个任务后自动认领下一个（tryClaimNextTask），leader 也可以根据进展重新分配。没有 Agent Teams 后，如何在 Redis MCP 或文件 IPC 的基础上实现动态调度？

包括：
- 根据任务类型自动选择合适的 agent（model 选择、tool 限制、system prompt）
- 任务完成后自动认领下一个
- leader 根据进展动态调整优先级或重新分配
- 新任务动态加入队列
- worker 数量的弹性伸缩（按需启动/关闭 session）

## Constraints
- 不修改 Claude Code 源代码
- 每个 claude -p 调用是独立进程，无状态
- Background agent 在 parent 进程内，共享内存但不能持续运行
- Redis 可以做任务队列（BRPOP）和 pub/sub 通知
- 外部 orchestrator 脚本可以管理 worker 生命周期

## Key Questions
- Worker pool 模式：预启动 N 个 session 等待任务，还是按需 spawn？
- 任务路由：怎么根据任务类型选择合适的 model/agent/tool set？
- 弹性伸缩：怎么决定何时增加/减少 worker？
- 状态感知：orchestrator 怎么知道每个 worker 的当前状态和能力？
- Agent Teams 的 tryClaimNextTask 是乐观锁——外部方案用什么替代？
