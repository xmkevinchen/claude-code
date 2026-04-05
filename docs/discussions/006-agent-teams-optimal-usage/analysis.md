---
id: "006"
title: "Analysis: Agent Teams 最优使用策略"
type: analysis
created: 2026-04-04
tags: [agent-teams, multi-agent, prompt-engineering, configuration]
---

# Analysis: Agent Teams 最优使用策略

## Question

从源代码来看，怎么样才是最高效的使用 Agent Teams 方式？如何保证 Session 主 Agent 在条件允许时优先使用 Agent Teams，只在不允许时才 fallback 到普通 subagents？

## Findings

### 1. Agent Teams vs 普通 Subagent 的架构区别

**决策分叉点**：`src/tools/AgentTool/AgentTool.tsx:262-316`

```
if (teamName && name) → spawnTeammate()  // Team member 路径
else                  → runAgent()        // 普通 subagent 路径
```

| 维度 | Agent Teams | 普通 Subagent |
|------|------------|--------------|
| 启动方式 | `spawnTeammate()` → tmux pane / in-process | `runAgent()` → async generator / fork |
| 生命周期 | 持久化，idle 后可唤醒 | 一次性，返回结果即销毁 |
| 通信 | `SendMessage` → mailbox 系统 | `<task-notification>` XML 回传 |
| 共享状态 | 共享 task list (`~/.claude/tasks/{team}/`) | 无共享状态 |
| 注册 | TeamFile (`~/.claude/teams/{team}/config.json`) | 无注册 |
| Prompt cache | 同 session 共享 prompt cache | Fork 模式共享，否则独立 |

**Coordinator Mode 是独立功能**（`src/coordinator/coordinatorMode.ts:36`）：
- `CLAUDE_CODE_COORDINATOR_MODE=1` 完全替换 system prompt，将主 agent 变为纯协调者
- 这与 Agent Teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`) 是正交的两个功能
- Coordinator 使用 `worker` subagent_type，不需要 Agent Teams 启用

### 2. 启用条件与硬约束

**启用 gate**（`src/utils/agentSwarmsEnabled.ts:24`）：

```
isAgentSwarmsEnabled() =
  (内部用户 → true)
  OR (CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 AND tengu_amber_flint killswitch=true)
```

**硬约束**：
1. **扁平拓扑**：teammates 不能再 spawn teammates（`AgentTool.tsx:272-273` 抛 Error）
2. **单 team 限制**：一个 leader 同时只能管理一个 team（`TeamCreateTool.ts:136-139`）
3. **In-process fallback**：无 tmux/iTerm2 时静默降级为 in-process，此时 teammates 不能 spawn background agents
4. **GrowthBook killswitch**：Anthropic 可远程禁用 `tengu_amber_flint`，用户无法控制

### 3. 什么时候 Agent Teams 值得开销

**值得用 Teams 的场景**：
- 2+ 个真正独立的并行子任务（每个 >30 tool calls）
- 需要角色专业化（researcher + coder + reviewer）
- 需要对抗性审查（challenger pattern）
- 长任务中 context 保持比 re-briefing 更便宜

**不值得用 Teams 的场景**：
- 顺序任务（team 增加延迟无并行收益）
- 单文件操作、简单查找（直接用 Glob/Grep/Read）
- 协调 token 开销超过工作 token（idle 通知、SendMessage、TaskList 读写）
- 子任务间高度耦合需要频繁同步

**Token 经济学**：Agent Teams 在某些场景下 token 消耗可达单 agent 的 ~7x（每个 teammate 独立 context + 协调开销）。但在真正的并行场景中，wall-clock time 显著缩短。

### 4. 让主 Agent "听懂人话" 的可靠性层级

**从高到低排列**：

#### Tier 1: Skills 作为入口（最可靠）
- `ae:team`、`ae:work`、`ae:analyze` 等 skill 展开后直接包含 `TeamCreate` 作为第一步
- 用户主动调用 → skill prompt 指令 LLM → LLM 执行 TeamCreate
- **LLM 不参与"是否用 Teams"的决策** — skill 已经替它决定了
- 推荐：复杂任务一律通过 skill 入口触发

#### Tier 2: 自定义 Agent 定义（可靠）
- `.claude/agents/*.md` 的 frontmatter 可限定 `tools` allowlist
- 定义一个 "team-coordinator" agent，只给 `TeamCreate/SendMessage/TaskCreate/TaskUpdate`
- `resolveAgentTools()` 硬编码执行 allowlist — 这是代码级强制

#### Tier 3: CLAUDE.md 条件规则（软偏好，部分可靠）
- CLAUDE.md 作为 user context 注入，不是 system prompt，权重低于 tool descriptions
- **具体条件规则** 比模糊偏好有效得多：

```markdown
## Delegation Policy
- 任务有 2+ 个独立并行子任务时，必须先 TeamCreate 再 spawn teammates
- 只有以下情况使用普通 Agent：单一、短小、自包含的查找/读取任务
- 如果不确定，默认 TeamCreate
```

- **弱点**：
  - 子 agent 可能设置 `omitClaudeMd: true`（如 Explore agent），看不到这些指令
  - 长对话中 CLAUDE.md 被压缩后效力下降
  - 模型可能判断"这个任务太简单不需要 team"而绕过

#### Tier 4: 权限系统强制（硬编码但过于激进）
- `settings.json` 的 `permissions.deny` 可以禁用 `Agent` 工具
- 这会完全阻止任何 subagent 调用，包括合理的场景
- **不推荐**：过于粗暴，杀敌一千自损八百

### 5. 推荐的配置方案

#### settings.json（已配置）
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

#### CLAUDE.md 推荐追加的 Delegation Policy

```markdown
## Agent Teams 使用策略

### 何时用 Agent Teams（TeamCreate → teammates → SendMessage）
- 任务有 2+ 个独立并行子任务
- 需要多角色协作（研究 + 实现 + 审查）
- 需要对抗性审查或跨家族意见
- 任务预期 >50 tool calls

### 何时用普通 Agent（不带 team_name）
- 单一、短小、自包含的查找任务
- 简单的文件搜索或代码探索（优先用 Glob/Grep/Read）
- 预期 <20 tool calls 的微任务

### 默认行为
- 不确定时，倾向 TeamCreate
- 所有 ae: skill 系列任务自动走 Agent Teams（由 skill 保证）
```

#### 运行环境建议
- 确保 tmux 或 iTerm2 可用，避免 in-process fallback 导致 teammate 能力受限
- 验证方式：检查 `getResolvedTeammateMode()` 是否返回 tmux/iterm2

### Challenges & Disagreements

**Challenger 的关键质疑**：

1. **"CLAUDE.md 是弱机制"** — 确认属实。CLAUDE.md 是 user context 而非 system prompt，对模型的约束力有限。但作为 Tier 3 补充手段仍有价值，不应期望它独立承担强制功能。

2. **"GrowthBook killswitch 不可控"** — 确认属实。`tengu_amber_flint` 是 Anthropic 的远程 killswitch，default=true 但随时可关。用户侧无解，只能接受这个风险。

3. **"扁平拓扑是硬限制"** — 确认属实。任何需要嵌套团队的工作流必须在 leader 层面拆分，不能依赖 teammate 递归 spawn。

4. **"Coordinator Mode ≠ Agent Teams"** — 确认属实且重要。两者是正交功能，不应混淆。Coordinator Mode 更适合"主 agent 完全不做实际工作"的场景，而 Agent Teams 适合"主 agent 作为 TL 兼做部分工作"的场景。

**Codex 的建议**：policy stack（env var + CLAUDE.md 条件规则 + 可选 permissions deny）是合理的分层方案。但 permissions deny Agent 过于激进，不推荐。

**Gemini 的建议**：Skills 作为入口是最可靠路径，因为它绕过了 LLM 的自主决策。对于需要保证 Agent Teams 使用的场景，应该封装成 skill。

## Summary

1. **Agent Teams 的最高效使用方式**：用于 2+ 个独立并行子任务、多角色协作、对抗性审查。避免用于顺序小任务。关键是判断"协调开销 < 并行收益"。

2. **让主 Agent 优先用 Agent Teams 的可靠路径**：
   - **最可靠**：通过 skills（`ae:team`/`ae:work`/`ae:analyze`）作为入口，skill 直接指令 TeamCreate
   - **次可靠**：自定义 `.claude/agents/` 定义，用 tools allowlist 硬编码
   - **辅助**：CLAUDE.md 中写具体条件规则（"2+ 并行子任务 → TeamCreate"），作为软引导
   - **不可靠**：模糊指令如"优先使用 Agent Teams" — 模型会在判断任务简单时绕过

3. **根本性限制**：没有原生的"prefer Agent Teams"开关。当前架构下，tool selection 是 LLM 概率性决策，只有 skills/自定义 agent/permissions 能提供代码级强制。CLAUDE.md 是必要但不充分的。

## Possible Next Steps

- `/ae:discuss 006` — 讨论具体的 CLAUDE.md delegation policy 措辞和自定义 agent 定义方案
- `/ae:plan` — 制定实施计划：编写 CLAUDE.md 策略 + 创建自定义 team-coordinator agent + 封装常用 skill
- `/ae:think` — 深度思考：是否应该 patch AgentTool prompt 加入 "prefer teams" 引导（需要修改源码）
