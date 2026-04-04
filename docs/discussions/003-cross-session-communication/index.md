---
id: "003"
title: "跨 Session 通讯方案"
status: active
created: 2026-04-04
pipeline:
  analyze: skipped
  discuss: done
  plan: pending
  work: pending
plan: ""
tags: [cross-session, ipc, redis, mcp, orchestration]
---

# 跨 Session 通讯方案

基于源代码，在不修改源代码的情况下，实现多个 Claude Code session 之间通讯和动态协作的可行方案。

## Topics

| # | Topic | Status | Decision |
|---|-------|--------|----------|
| 1 | 可用的跨 session IPC 原语 | converged | 4 个原语：FileChanged + asyncRewake + additionalContext + 文件系统 |
| 2 | 基础通讯架构方案 | converged | 3 个方案：文件信号、watchdog、cron 轮询 |
| 3 | Redis MCP 阻塞订阅问题 | converged | MCP tool 禁止阻塞操作，BRPOP timeout=1 可接受 |
| 4 | Listener 模式设计 | converged | asyncRewake + redis-cli 首选，外部 daemon 最可靠 |
| 5 | Redis vs 文件 IPC 取舍 | converged | 共存，按场景选择，不做静默 fallback |
| 6 | 部署与配置复杂度 | converged | Docker one-liner + uvx MCP server |
| 7 | 动态任务分配与队员调度 | converged | 中心化编排器 + queue-per-model + BRPOP 原子认领 |

## Documents
- [Conclusion — 基础 IPC 方案](conclusion.md)
- [Conclusion — Redis + 动态调度](conclusion-redis.md)
