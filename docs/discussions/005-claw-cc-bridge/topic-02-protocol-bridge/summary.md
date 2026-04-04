---
id: "02"
title: "协议桥接：MCP 作为通用语言"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 协议桥接

## Current Status
待讨论。

## Context
MCP（Model Context Protocol）是两侧都支持的开放标准。Claude Code 有完整的 MCP client。OpenClaw 计划支持 MCP。如果 Claw 实例作为 MCP server 暴露自己，Claude Code 就可以原生调用它。反过来，如果 Claude Code session 暴露为 MCP server（不现实——没这个能力），或者 Claw 通过其他方式注入消息到 CC。

## Constraints
- MCP 是 client-server 模型（不是 peer-to-peer）
- Claude Code 只能做 MCP client（不能做 server，除非 feature('DIRECT_CONNECT') 在外部构建可用——已确认不可用）
- MCP tool 调用是同步的
- Claw 可以同时做 MCP client 和 server

## Key Questions
- Claw 作为 MCP server + Claude Code 作为 MCP client 是否可行？
- MCP 的同步调用限制怎么处理？（CC 调 Claw MCP tool 会阻塞 CC 的 turn）
- 反向通道（Claw→CC）用什么？MCP 没有 server→client 推送
- 有没有一个统一的桥接架构能同时解决双向通讯？
