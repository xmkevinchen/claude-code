---
id: "02"
title: "原生通讯架构"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 原生通讯架构

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Claude Code 的 Agent Teams 通讯是事后加的——in-process AsyncLocalStorage 隔离 + 文件 mailbox。设计缺陷：isActive 状态不可靠、shutdown 依赖模型响应、无超时强制、无 heartbeat。

OpenClaw 可以从第一天就把通讯作为核心能力设计进去，而不是 bolt-on。今天研究的方案：
- 文件 IPC（已验证，零依赖）
- Redis pub/sub + 队列（低延迟，原子认领）
- asyncRewake hook（推送唤醒）
- 外部编排器

核心问题：claw 实例是独立进程还是同进程？通讯走内存还是 IPC？

## Constraints
- 必须支持真正的多进程（不是 AsyncLocalStorage 伪隔离）
- 一个 claw 崩溃不能影响其他 claw
- 通讯延迟要可接受（<1s）
- 需要支持 1:1、1:N、N:N 通讯模式
- 不能依赖特定云服务（本地 first）

## Key Questions
- 传输层：Unix socket / TCP / Redis / 文件？哪个作为默认？
- 消息协议：JSON-RPC？protobuf？plain JSON lines？
- 发现机制：claw 实例怎么找到彼此？注册中心？广播？约定路径？
- 通讯原语：send/receive/broadcast/subscribe 够了吗？需要 request/response 模式吗？
- 与 MCP 的关系：通讯层要不要复用 MCP 协议？claw-to-claw 通讯本质上类似 MCP server-to-server
