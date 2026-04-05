---
id: "02"
title: "产品/项目点子"
status: pending
current_round: 1
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 产品/项目点子

## Current Status
待团队调查

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Claude Code 的架构中有很多可以被独立复用或启发新项目的模式：自定义 React terminal renderer（不依赖 Ink npm）、MCP server 发现和桥接、多 Agent 协调、permission 系统、prompt caching 策略、context compaction、IDE bridge（WebSocket）、auto memory/dreaming 等。从这些实现中可以提取什么独立的产品或工具？

## Constraints
- 需要区分"可以独立复用"和"深度耦合 Claude API 无法分离"的部分
- 法律/IP 限制：源码从 npm sourcemap 提取，衍生作品需注意许可证

## Key Questions
- 哪些模块有独立价值，可以抽取为开源工具或库？
- Claude Code 的哪些设计模式可以启发竞品或互补产品？
- 用户（开发者社区）最可能需要什么衍生工具？
