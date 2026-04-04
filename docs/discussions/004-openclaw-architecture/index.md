---
id: "004"
title: "OpenClaw 精简版架构"
status: active
created: 2026-04-04
pipeline:
  analyze: skipped
  discuss: done
  plan: pending
  work: pending
plan: ""
tags: [openclaw, architecture, multi-agent, ipc, go]
---

# OpenClaw 精简版架构

基于 Claude Code 源码设计，Go 重写，原生支持多 claw 实例间通讯，model-agnostic。

## Topics

| # | Topic | Status | Decision |
|---|-------|--------|----------|
| 1 | 核心提取 | converged | Tool 接口 35->8 方法，2 层 compact，保留 hooks + MCP |
| 2 | 原生通讯 | converged | P2P mesh over UDS，JSON-RPC 2.0，heartbeat + 超时强杀 |
| 3 | Model-agnostic | converged | 内部 adapter 接口 + 3 原生 adapter (Claude/OpenAI/Ollama) |
| 4 | MVP 范围 | converged | Go, 10 tools (含 claw_send/claw_request), <5K 行 |
| 5 | 技术栈 | converged | Go（单二进制、goroutine IPC、3 周 MVP） |
| 6 | 跨设备通讯 | converged | Transport 抽象：v0.1 UDS -> v0.2 TCP -> v0.3 NATS |

## Documents
- [Conclusion](conclusion.md)
- [Claw <-> CC 桥接](../005-claw-cc-bridge/conclusion.md)
