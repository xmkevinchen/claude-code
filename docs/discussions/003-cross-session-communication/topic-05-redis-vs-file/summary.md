---
id: "05"
title: "Redis vs 文件 IPC 取舍"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: Redis vs 文件 IPC 取舍

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
已经有一套验证通过的文件 IPC 方案（session-ipc-*.sh 脚本）。Redis MCP 增加了 Docker 依赖和 MCP server 维护成本。需要评估什么场景下 Redis 的优势（低延迟、原生 pub/sub、多播）值得额外复杂度。

## Constraints
- 文件 IPC 已验证可用（init/send/watch/list 脚本全部通过）
- Redis 需要 Docker + Python MCP server
- 两种方案可以共存（不互斥）

## Key Questions
- 什么具体场景下文件 IPC 不够用，必须上 Redis？
- Redis 的延迟优势（<1ms vs 1s 轮询）在实际 Claude Code 工作流中有多大意义？
- 维护成本：Docker 容器 + MCP server 的长期可靠性如何？
- 有没有"文件 IPC + Redis 混合"的方案——简单场景用文件，高频场景用 Redis？
