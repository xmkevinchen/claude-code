---
id: "04"
title: "负面螺旋及其他风险：如何在 MVP 中验证"
status: pending
current_round: 1
created: 2026-04-05
decision: ""
rationale: ""
reversibility: ""
---

# Topic: 负面螺旋和风险验证

## Current Status
待团队讨论

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Product vision 列了 4 个风险：garbage in、冷启动、隐私、闭环负面螺旋。其中负面螺旋最危险——早期摄入的错误知识被 AI 引用后产出基于错误前提的新知识，错误自我强化。

MVP Phase 1 是验证闭环的阶段。需要在这个阶段就设计验证机制，不能等到 Phase 3 才发现闭环根本不 work。

## Constraints
- MVP 阶段资源有限，不可能做完善的 guardrail
- 但必须有足够的可观测性来发现问题
- 负面螺旋的检测本身就是一个有价值的功能（不只是风险缓解）

## Key Questions
- 负面螺旋在实践中到底会不会发生？什么条件下最容易发生？
- MVP 阶段用什么机制检测负面螺旋？
- 矛盾检测在 MVP 中做到什么程度够用？
- 冷启动问题有没有更好的解法？（比如直接导入 AE 已有的 discussions）
- 怎么设计 MVP 的 metrics 来判断"闭环是正面螺旋还是负面螺旋"？
