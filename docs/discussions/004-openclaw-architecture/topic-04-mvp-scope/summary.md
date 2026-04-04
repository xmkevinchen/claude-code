---
id: "04"
title: "MVP 范围与技术栈"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: MVP 范围与技术栈

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
需要定义 OpenClaw v0.1 的最小可行范围——能跑起来、能展示核心价值（多 claw 通讯），但不过度工程。

技术栈选择：Claude Code 用 TypeScript + Bun + React/Ink。OpenClaw 不一定要跟随。考虑因素：生态、性能、开发效率、社区贡献友好度。

## Constraints
- MVP 要在合理时间内完成（一个人或小团队）
- 核心价值是多 claw 通讯 + model-agnostic，不是全功能 CLI
- 需要足够好的 DX 让社区愿意贡献
- 需要可扩展（plugin/tool 系统），但 MVP 不需要完整 plugin marketplace

## Key Questions
- 技术栈：TypeScript (Bun/Node) vs Rust vs Go vs Python？哪个平衡开发效率和性能？
- MVP 包含哪些 tools？最小集是什么？
- UI：全屏 TUI (ratatui/ink) vs 简单 readline vs headless-first？
- 打包分发：npm publish vs 单二进制 vs both？
- 开源策略：MIT？Apache 2.0？怎么吸引贡献者？
