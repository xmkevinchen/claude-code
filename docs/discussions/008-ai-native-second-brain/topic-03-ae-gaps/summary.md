---
id: "03"
title: "AE plugin 当前 gap：闭环的阻碍"
status: pending
current_round: 1
created: 2026-04-05
decision: ""
rationale: ""
reversibility: ""
---

# Topic: AE plugin 当前 gap

## Current Status
待团队讨论

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
要实现"AE 产出 → Second Brain 摄入 → 反哺下次 AE"的闭环，AE plugin 当前有什么 gap？

已知的问题（来自 027 源码审计）：
- SendMessage 到 "TL" 而非 "team-lead"（通信问题）
- Pre-check 错误 key path（配置问题）
- 无 Agent Teams 时全部 skill 不可用（无降级）
- dev→qa lateral messaging 违反 protocol

除了这些 bug fix 之外，AE 缺什么才能支持 Second Brain 闭环？

## Constraints
- AE 的 skill 定义是 Markdown（prompt template），不是代码——修改成本低
- AE 的输出已经是结构化 Markdown，摄入端不需要 AE 改
- 反哺端（下次 AE 运行时注入 Second Brain 上下文）可能需要 AE skill 改动

## Key Questions
- AE 的哪些 skill 需要改动来支持"从 Second Brain 读取上下文"？
- 改动量有多大？是加一行 prompt 还是要重构 flow？
- AE 的输出格式有没有需要为 Second Brain 摄入做的调整？
