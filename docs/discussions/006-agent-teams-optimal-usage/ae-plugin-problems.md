---
id: "006-b"
title: "Analysis: AE Plugin Agent Teams 问题诊断"
type: analysis
created: 2026-04-04
tags: [ae-plugin, agent-teams, bugs, protocol-violations]
---

# Analysis: AE Plugin Agent Teams 问题诊断

## Question

根据 Claude Code 源码分析的结果，AE plugin 在使用 Agent Teams 时存在哪些问题？

## Findings

### Critical Issues (P1)

#### 1. SendMessage 到 "TL" 消息投递失败 — 全部 skill 受影响

**严重度**：Critical — 所有 team-based skill 的核心通信路径断裂

Claude Code 源码中，team lead 的 mailbox 名称由 `TEAM_LEAD_NAME = 'team-lead'`（`src/swarm/constants.ts:1`）定义。`useInboxPoller`（`useInboxPoller.ts:97-102`）轮询 lead 注册名下的 mailbox。

但 AE 所有 skill 的 agent prompt 都写 `SendMessage findings to Lead (TL)` 或 `SendMessage to TL`。当 agent 发送消息给 "TL" 时：
1. `SendMessageTool` 在 `agentNameRegistry` 中找不到 "TL"
2. Fallthrough 到 `handleMessage()` → `writeToMailbox("TL")`
3. Lead 轮询的是 "team-lead" mailbox，不是 "TL" mailbox
4. **消息丢失，永远不被读取**

**受影响 skill**：ae:analyze、ae:think、ae:discuss、ae:plan、ae:review、ae:work、ae:consensus、ae:team — 几乎全部

**修复**：所有 skill 中 `SendMessage findings to Lead (TL)` → `SendMessage findings to team-lead`

> **Challenger 质疑**：实际运行中 AE plugin 显然是工作的（用户在用），说明 Claude Code 的消息投递可能有 fallback 路径（例如 in-process teammates 的消息不经过 mailbox 而是直接进入 parent context）。需要实测验证。但如果依赖的是 undocumented fallback，这仍是脆弱的。

#### 2. Pre-check 错误消息使用错误的 key path — 10+ 个 skill 受影响

**严重度**：High — 阻塞首次配置的用户

所有 skill 的 pre-check 指令用户添加：
```json
{ "experiments": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": true } }
```

但 Claude Code settings.json 的 schema 中 **没有 `experiments` key**。正确的路径是：
```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

证据：
- `src/utils/managedEnv.ts:136-190`：env vars 从 `settings.env` 读取
- `managedEnvConstants.ts:141`：`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 在 `SAFE_ENV_VARS` 中
- `agentSwarmsEnabled.ts:32`：通过 `isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS)` 检查

**受影响 skill**（所有带 pre-check 的）：
`analyze:13, consensus:15, think:14, plan:23, review:21, work:48, team:15, discuss(implicit), trace:14, testgen:14, plan-review:20`

**修复**：全部 skill 的 pre-check error message 改为 `"env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }`

#### 3. 无 Agent Teams 时全部 skill 不可用 — 无降级路径

**严重度**：High — 单点故障

所有 team-based skill 在 pre-check 中硬性要求 Agent Teams。如果不可用（env var 未设、GrowthBook killswitch `tengu_amber_flint` 关闭），**整个 AE plugin 变成废铁**。

- `ae:work` 即使是 "Single-platform step → Lead executes TDD cycle directly"（不需要 team），仍然被 Check 3 拦住
- `ae:code-review` 实际上不用 TeamCreate（4 个 track 都是独立 subagent），但如果有 pre-check 也会被拦
- `ae:think` 和 `ae:analyze` 的核心价值是研究，完全可以用单 agent 降级执行

**修复建议**：
1. 区分"需要 Agent Teams"的 skill（ae:discuss、ae:review、ae:consensus）和"可以降级"的 skill
2. 可降级 skill 在 Agent Teams 不可用时 fallback 到 plain Agent subagent 模式
3. `ae:work` 的 Check 3 应该仅在 parallel mode 时检查

### High Issues (P2)

#### 4. dev→qa 直接通信违反 Base Protocol

`ae:work/SKILL.md:77-83` 中 dev agent prompt 写 `SendMessage to qa when done`，qa 写 `Send findings to dev, wait for fixes, re-review`。

这是 teammate→teammate lateral messaging，直接违反 `ae:agent-teams` Base Protocol：
> "Agents SendMessage to TL only. No lateral agent-to-agent messages unless TL explicitly routes a debate exchange."

**影响**：
- TL 失去对 dev→qa 交接的控制（无法验证 dev 是否真的完成）
- 多 dev 并行时 qa 收到信号顺序不确定
- 消息可能因 mailbox 路由问题丢失

**修复**：dev → TL（"Step N complete, ready for QA"）→ TL 验证 → TL → qa

#### 5. standards-expert "等待 archaeologist" 是不可能完成的指令

`ae:analyze/SKILL.md:76-80` 指令 standards-expert："Wait for archaeologist's code analysis before comparing。"

但两者同时以 `run_in_background: true` spawn。standards-expert 没有任何机制知道 archaeologist 何时完成：
- 没有 shared state
- 没有 TL 路由的触发信号
- 没有 polling 机制

**实际行为**：standards-expert 要么忽略等待指令自行研究（等同于无依赖），要么死等（deadlock/timeout）。

**修复**：TL 应在收到 archaeologist findings 后，主动 SendMessage 给 standards-expert 附带 archaeologist 的关键发现。这才是 "TL routes all information" 原则的正确实现。

#### 6. ae:plan Doodlestein 在 team 外运行 — 挑战无法路由回审查者

`ae:plan/SKILL.md` Step 3 末尾 "Close the Team"，Step 4 Doodlestein 用独立 subagent（无 team_name）。

Doodlestein Protocol 要求：challenges 路由给原始 team members → 他们 respond → TL 判断 validity。但 Step 4 时 plan-review team 已关闭，architect 和 dependency-analyst 已 shutdown。Doodlestein challenges 只能呈现给 user。

这削弱了 Doodlestein 的对抗强度。ae:discuss 正确地在 team 内运行 Doodlestein（Step 7），ae:plan 应该效仿。

**修复**：将 Step 4 移到 Step 3 "Close the Team" 之前，Doodlestein agents spawn 进现有 team。

### Medium Issues (P3)

#### 7. "Follow Team Communication Protocol" 可能是悬空引用

所有 spawn prompt 中的 `Follow Team Communication Protocol` 依赖 `subagent_type` 映射的 agent 定义文件被注入 agent context。如果定义文件不被注入（in-process spawning 可能只用 name hint），这个短语就是无效指令。

需要实测验证 agent 定义文件是否在 spawn 时注入。

#### 8. `model: inherit` 非标准值

Doodlestein agents（`doodlestein-strategic.md`、`doodlestein-adversarial.md`、`doodlestein-regret.md`）frontmatter 中 `model: inherit`。Claude Code 标准 model 值是 sonnet/haiku/opus 或完整 model ID。`inherit` 可能被忽略（静默 fallback 到 session default）或报错。

#### 9. Session 中断打破 "one team, one lifecycle"

`ae:discuss` 要求 team 从 Step 2 持续到 Step 9（Conclusion）。但 session 中断后，team state 无法恢复。Skill 说 "If the team already exists (resuming), skip to step 3" 但没有 team_id 或 team_state 持久化机制。

实际效果：中断后 spawn 全新 team，丢失消息历史和 agent context。文件系统（summary.md、round-NN.md）是唯一 bridge。

#### 10. 扁平命名空间 — agent name 冲突风险

`ae:work` 中 `Agent(name: "qa", ...)` 如果用户项目有同名 custom agent（`.claude/agents/qa.md`），agent-selection 规则说 "prefer project agents"，可能导致 spawn 了不同于预期的 agent。Team 内 name 路由无命名空间隔离。

#### 11. 120s proxy timeout 仅为约定，无强制执行

`ae:agent-selection` 定义 120s 超时，但 Claude Code 无 hard timeout on teammate execution。如果 MCP proxy 挂起，team 无限等待。依赖 agent 自行报告 timeout。

### Design Observations（非 Bug）

#### 12. ae:analyze 和 ae:think 高度重叠

两者仅 1 个 core agent 不同（archaeologist vs architect），flow 几乎相同。差异化价值主要来自 prompt framing。从可维护性角度是重复代码。

#### 13. 每个 skill 独立创建/销毁 team — 无跨 skill context 复用

连续运行 analyze→discuss→plan→work 时，agents 被反复创建销毁。文件系统是唯一 context bridge。这是架构取舍（single-team-per-leader 限制），改进方向是更精确的 artifact handoff 而非跨 skill team 复用。

#### 14. ae:code-review 不用 TeamCreate — 混合模式

Tracks 1-4 全部是独立 subagent（无 team_name），与其他 skill 的 team-based 模式不一致。如果将来需要 track 间协调，需重构。

## Summary

| # | Issue | Severity | Category |
|---|-------|----------|----------|
| 1 | SendMessage 到 "TL" 而非 "team-lead" | Critical | 通信断裂 |
| 2 | Pre-check 错误 key path（experiments → env） | High | 配置阻塞 |
| 3 | 无 Agent Teams 时全部 skill 不可用 | High | 无降级 |
| 4 | dev→qa lateral messaging 违反 protocol | High | 协议违规 |
| 5 | "等待 archaeologist" 无协调机制 | High | Race condition |
| 6 | ae:plan Doodlestein 在 team 外运行 | High | 对抗强度削弱 |
| 7 | "Follow Team Communication Protocol" 悬空引用 | Medium | 需验证 |
| 8 | `model: inherit` 非标准值 | Medium | 可能静默失败 |
| 9 | Session 中断打破 team lifecycle | Medium | 持久化缺失 |
| 10 | Agent name 扁平命名空间冲突 | Medium | 命名冲突 |
| 11 | Proxy timeout 无强制执行 | Low | 挂起风险 |

**核心结论**：AE plugin 的最严重问题是 Issue 1（SendMessage 到 "TL"）和 Issue 2（pre-check key path），这两个直接影响功能正确性。Issue 3（无降级路径）影响可用性范围。Issues 4-6 是协议设计层面的不一致，影响可靠性但不完全阻塞功能。

## Possible Next Steps

- `/ae:plan` — 制定修复计划，优先修复 P1（SendMessage target name + pre-check key path + degradation path）
- `/ae:discuss` — 讨论是否需要重构 ae:work 的 dev→qa 通信模式和 ae:plan 的 Doodlestein 时机
- 实测验证 Issue 1 和 Issue 7 — 在实际 Agent Teams session 中确认 "TL" 是否真的导致消息丢失
