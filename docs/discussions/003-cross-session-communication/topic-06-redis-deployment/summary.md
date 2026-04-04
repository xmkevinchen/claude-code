---
id: "06"
title: "部署与配置复杂度"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 部署与配置复杂度

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Redis MCP 方案需要：Docker 运行 Redis、Python MCP server 脚本、.mcp.json 注册、每个 session 都能访问同一个 Redis 实例。需要评估这套基建的安装、配置、维护成本，以及是否有更轻量的替代（如 SQLite、named pipe）。

## Constraints
- macOS 环境（Docker Desktop 或 colima）
- MCP server 需要注册到 .mcp.json 或 settings.json
- 所有参与通讯的 session 必须能连接到同一个 Redis
- 需要考虑 Redis 挂了的 fallback

## Key Questions
- 最小安装步骤是什么？能不能一条命令搞定？
- MCP server 用 uvx/npx 免安装还是需要本地 pip install？
- Redis 挂了怎么办？需要 fallback 到文件 IPC 吗？
- 有没有比 Redis 更轻量的消息中间件（SQLite WAL、Unix named pipe、macOS XPC）？
