---
id: "03"
title: "Redis MCP 阻塞订阅问题"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: Redis MCP 阻塞订阅问题

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
MCP tool 调用是同步的——模型调用一个 tool，等待返回，才能继续。Redis SUBSCRIBE/BRPOP 是阻塞操作。如果 MCP tool 实现了阻塞等待，那个 turn 就被卡住了，模型无法做其他事。这是 Redis MCP 方案的核心技术挑战。

## Constraints
- 不修改 Claude Code 源代码
- MCP tool 调用是同步的（模型等待 tool 返回）
- Background agent 可以在后台运行（run_in_background: true）
- Background agent 完成后通过 <task-notification> 通知 parent
- asyncRewake hook 可以后台运行并唤醒模型（exit 2）

## Key Questions
- MCP tool 的阻塞调用会不会导致整个 session 卡死？
- 用 background agent 做 subscriber，每次只等一条消息然后退出，是否可行？
- 有没有非阻塞的替代方案（比如 LPUSH/RPOP 轮询）？
- MCP tool 有没有 timeout 机制防止无限阻塞？
