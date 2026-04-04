---
id: "005"
title: "Claw <-> Claude Code Session 通讯"
status: active
created: 2026-04-04
pipeline:
  analyze: skipped
  discuss: done
  plan: pending
  work: pending
plan: ""
tags: [openclaw, claude-code, bridge, ipc, mcp]
---

# Claw <-> Claude Code Session 通讯

OpenClaw 实例与 Claude Code session 之间的双向通讯方案。

## Topics

| # | Topic | Status | Decision |
|---|-------|--------|----------|
| 1 | 桥接机制 | converged | CC->Claw: MCP tool call; Claw->CC: file + asyncRewake |
| 2 | 协议桥接 | converged | Claw UDS 双人格（JSON-RPC + MCP server），无 daemon |

## Documents
- [Conclusion](conclusion.md)
