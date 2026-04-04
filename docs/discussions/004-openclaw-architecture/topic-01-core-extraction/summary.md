---
id: "01"
title: "核心提取：保留什么砍什么"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 核心提取：保留什么砍什么

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Claude Code 的架构分层（从今天的源码分析）：
- Core: QueryEngine + query loop + Tool 系统 + buildTool factory
- Tools: 40+ tools (Bash, FileRead/Write/Edit, Glob, Grep, Web, Agent, MCP...)
- Services: API client, MCP client, compact, cost tracker, autoDream, OAuth
- UI: 自定义 Ink renderer (251K 行), React components
- Swarm: TeamCreate/SendMessage/TaskList/Mailbox/idle-wake
- Anthropic-specific: GrowthBook, undercover, buddy, KAIROS, enterprise MDM, referral
- Bridge: IDE integration (VS Code, JetBrains)

OpenClaw 需要确定哪些是核心（必须有），哪些是增值（可以后加），哪些直接砍掉。

## Constraints
- 目标是精简——不是复制 Claude Code 的所有功能
- 核心价值是 tool 系统 + 多 claw 通讯，不是 UI
- 需要 model-agnostic（API client 不能绑 Claude）
- MCP 协议是开放标准，应该保留
- 不需要 IDE bridge（至少 MVP 不需要）

## Key Questions
- Tool 系统的 buildTool factory + ToolDef 模式要保留还是简化？35 个方法的 Tool 接口要不要瘦身？
- 自定义 Ink renderer（251K 行）是否值得保留？还是用标准 Ink 或者更简单的 CLI 框架？
- 哪些 services 是核心的（compact、cost tracker）？哪些是 Anthropic 绑定的（autoDream、OAuth）？
- 权限系统（三层 allow/deny/ask）要保留多少？
- Hooks 系统要保留吗？asyncRewake 是通讯的关键原语
