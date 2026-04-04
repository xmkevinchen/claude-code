---
id: "01"
title: "桥接机制"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 桥接机制

## Current Status
待讨论。

## Context
两侧的 IPC 能力已经完全清楚：

**OpenClaw 侧（我们设计的）：**
- UDS server：每个 claw 监听 ~/.openclaw/sockets/{name}.sock
- JSON-RPC 2.0 协议
- claw_send / claw_request 内置 tool
- registry.json 发现机制

**Claude Code 侧（源码验证的）：**
- asyncRewake hook（后台运行，exit 2 唤醒模型）
- FileChanged hook（检测文件变化）
- additionalContext 注入（hook JSON output）
- 文件系统读写（任何 session 都可以）
- MCP tool 调用（同步，默认 27.8h timeout）
- Bash tool（可以执行任意命令）
- Agent Teams SendMessage（如果 feature 可用）
- Print mode（claude -p，headless 单轮执行）

## Constraints
- 不修改 Claude Code 源代码
- Claude Code 侧的 MCP tool 是同步阻塞的
- OpenClaw 侧完全自主设计
- 需要双向通讯（不只是单向）

## Key Questions
- Claw→CC：最佳通道是什么？文件+asyncRewake？MCP？Bash 连 UDS？
- CC→Claw：最佳通道是什么？Bash socat？MCP server？文件？
- 能不能让 Claw 暴露为 MCP server，CC 直接调 MCP tool 与 Claw 通讯？
- 能不能让 CC 暴露为 Claw 的 adapter（CC 作为一个特殊的 "model"）？
