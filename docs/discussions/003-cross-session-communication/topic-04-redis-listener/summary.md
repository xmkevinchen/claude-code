---
id: "04"
title: "Listener 模式设计"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: Listener 模式设计

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
跨 session 通讯需要一个"监听者"——持续等待消息并处理。Agent Teams 用 idle/wake + 500ms 轮询实现。没有 Agent Teams 后，需要设计一个替代的 listener 模式。候选方案：background agent 做 subscriber、asyncRewake hook 做 watcher、cron 轮询、外部 daemon。

## Constraints
- Background agent 是一次性的——完成后退出，不能持续监听
- asyncRewake hook 也是一次性——exit 2 后进程退出
- Cron 最小粒度 1 分钟
- 外部 daemon 需要额外进程管理

## Key Questions
- 如何实现"持续监听"——每次只收一条消息就重启 listener？还是有更好的模式？
- Background agent + subscribe_once 循环是否可行？（agent 内部循环调用 MCP tool）
- asyncRewake hook + Redis SUBSCRIBE 是否可行？（hook 内部连接 Redis）
- 哪种 listener 模式最可靠、最省 token？
