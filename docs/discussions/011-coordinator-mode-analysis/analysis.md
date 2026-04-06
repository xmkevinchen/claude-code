---
id: "011"
title: "Analysis: Coordinator Mode Deep Dive"
type: analysis
created: 2026-04-05
tags: [coordinator-mode, swarm, agent-teams, multi-agent, orchestration]
---

# Analysis: Coordinator Mode Deep Dive

## Question

深挖 coordinator mode 的完整实现：1) coordinatorMode.ts 的核心逻辑和 swarm coordination 机制 2) 它和 Agent Teams 的关系 3) 和 /batch 的 prompt-only 编排的本质区别 4) AE plugin 可以从中学到什么

---

## 1. Coordinator Mode 核心实现

### 激活路径

```
isCoordinatorMode() — coordinatorMode.ts:36-41
  ├─ feature('COORDINATOR_MODE')  — Bun DCE 编译时 flag
  └─ process.env.CLAUDE_CODE_COORDINATOR_MODE  — 运行时 env var
      两者同时为真才激活
```

激活后的影响链：
- `systemPrompt.ts:62-75` — `buildEffectiveSystemPrompt()` 完全替换默认 system prompt 为 coordinator 专用 250 行 prompt
- `toolPool.ts:72-76` — `applyCoordinatorToolFilter()` 限制工具为 4 个 + MCP PR 订阅
- `QueryEngine.ts:304-308` — 注入 `workerToolsContext`（worker 工具列表 + MCP 服务器 + scratchpad 路径）

### 工具集对比

| 角色 | 工具集 | 来源 |
|------|--------|------|
| **Coordinator** | Agent, SendMessage, TaskStop, SyntheticOutput + MCP `*subscribe_pr_activity` | `COORDINATOR_MODE_ALLOWED_TOOLS` (tools.ts:107) |
| **Worker** (标准) | FileRead, WebSearch, Grep, Glob, Bash, FileEdit, FileWrite, NotebookEdit, Skill, ToolSearch, EnterWorktree, ExitWorktree | `ASYNC_AGENT_ALLOWED_TOOLS` (tools.ts:55) |
| **Worker** (SIMPLE) | Bash, FileRead, FileEdit | `CLAUDE_CODE_SIMPLE` env var 限制 |
| **In-process Teammate** | TaskCreate/Get/List/Update, SendMessage, CronCreate/Delete/List | `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` (tools.ts:77) |

**核心设计：Coordinator 不能读写文件、不能运行命令。** 所有实际操作必须通过 worker 完成。

### 结构化工作流（System Prompt 定义）

```
Phase 1: Research     — 并行 worker 调查代码库
Phase 2: Synthesis    — Coordinator 综合 worker 发现，生成精确 spec
Phase 3: Implementation — Worker 按 spec 实现
Phase 4: Verification — 独立 worker 验证（新鲜视角）
```

### 核心原则："Never Delegate Understanding"

```
coordinatorMode.ts:257-259:
"Never write 'based on your findings' or 'based on the research.'
 These phrases delegate understanding to the worker instead of
 doing it yourself."
```

Coordinator 必须阅读 worker 输出、理解问题、然后生成包含具体文件路径和行号的实现 spec。不允许模糊转发。

**Challenger 的关键质疑**：Coordinator 没有文件工具，无法验证 worker 报告的文件路径是否正确。它的 "synthesis" 实际上是 **对 worker 文本描述的改写，不是对代码的理解**。它把 "based on your findings, fix it" 变成了 "fix src/auth/validate.ts:42" — 但两者同样盲目。

**评估**：Challenger 的质疑成立但不完全。Coordinator 的 synthesis 确实不能验证 worker 的声明，但它提供了一个信息压缩和重组的步骤，减少了 cascading hallucination 的风险（MAST paper, arXiv 2503.13657）。这比完全盲目转发好，但不如有文件访问权限的 synthesis。

### Worker 生命周期

```
Worker 生成：
  AgentTool({ subagent_type: "worker", prompt: "..." })
    → builtInAgents.ts:35-42 解析为 coordinator-specific agent 定义
    → runAgent() 标准执行（非 teammate 路径）
    → 异步运行，完成后发送 <task-notification> XML

Worker 延续 vs 新建：
  高上下文重叠 → SendMessage 继续同一 worker
  低上下文重叠 → 新建 worker
  验证 → 必须新建（新鲜视角）
  错误修复 → 继续（保留错误上下文）
  错误方向 → 新建（避免上下文污染）
```

### Scratchpad 系统

- 路径：`/tmp/claude-{uid}/{sanitized-cwd}/{sessionId}/scratchpad/`
- Feature gate：`tengu_scratch`（Statsig）
- 权限绕过：`isScratchpadPath()` 跳过正常写入权限检查
- 用途：跨 worker 知识共享的共享文件系统目录
- Worker 可以在此读写而不触发权限确认

### Session Resume

`matchSessionMode()` (coordinatorMode.ts:49-78) 在 session resume 时翻转 env var 以匹配保存的 session 模式。**注意**：直接修改 `process.env`，在多 session 同进程场景下存在竞态条件风险。

---

## 2. Coordinator Mode vs Agent Teams：关系

### 结论：执行层共享，协议层独立

Archaeologist 称 "完全独立"，Challenger 反驳说底层共享 AgentTool 和 SendMessageTool。综合评估：

| 维度 | Coordinator Mode | Agent Teams |
|------|-----------------|-------------|
| **激活方式** | Feature flag + env var | `TeamCreateTool` 调用 |
| **System prompt** | 完全替换为 coordinator prompt | 不替换（保留默认） |
| **工具限制** | 只有 4 个工具 | 全工具集 |
| **Worker 类型** | `subagent_type: "worker"`（transient） | Named teammates（persistent） |
| **通信方式** | Task-notification XML（user-role message） | File-based mailbox（`~/.claude/mailbox/`） |
| **Worker 生命周期** | 完成即结束，可通过 SendMessage 继续 | 持续存在，idle 后等待消息 |
| **进程模型** | 单进程 in-process subagent | 可跨进程（tmux/iTerm2 pane）或 in-process |
| **状态文件** | 无 team file | `~/.claude/teams/{name}/config.json` |
| **共享基础设施** | AgentTool, SendMessageTool, runAgent() | AgentTool, SendMessageTool, runAgent() |

**本质区别**：
- **Coordinator Mode** = 受限的编排人格。Coordinator 只能指挥，不能亲自动手。单会话内的 manager-worker 模式。
- **Agent Teams** = 交互式多 agent 集群。所有 agent 都有完整工具，通过 mailbox 通信，可以跨终端 pane 运行。

Challenger 正确指出：两者共享执行基础设施（AgentTool 的 bug 会同时影响两个系统），但协议层（通信方式、生命周期、工具权限）确实独立。

**为什么两个系统共存？** 没有官方文档解释。从代码证据推断：
- Coordinator Mode 更早出现（codename `tengu`），是产品级 orchestrator 功能
- Agent Teams 后来作为交互式 swarm 能力加入（`tengu_amber_flint` gate）
- `INTERNAL_WORKER_TOOLS`（coordinatorMode.ts:29-34）显式阻止 worker 使用 TeamCreate/TeamDelete — 刻意隔离两个系统
- 两者面向不同用户需求：coordinator 面向 "管理者" 式自动化，Agent Teams 面向 "团队" 式协作

---

## 3. Coordinator Mode vs /batch：Runtime vs Prompt Coordination

| 维度 | Coordinator Mode | /batch |
|------|-----------------|--------|
| **协调方式** | Runtime — system prompt 替换 + 工具限制 | Prompt-only — 注入编排 prompt，不改工具集 |
| **工具限制** | Coordinator 只有 4 个工具 | Coordinator（即用户 session 的 LLM）有全部工具 |
| **Phase 结构** | System prompt 内置 Research→Synthesis→Implementation→Verification | Prompt 指导 Phase 1 (Plan) → Phase 2 (Spawn) → Phase 3 (Track) |
| **Worker 类型** | `subagent_type: "worker"`（coordinator-specific） | `subagent_type: "general-purpose"` + `isolation: "worktree"` |
| **Synthesis** | 强制 — coordinator 必须综合后才能指派实现 | 无强制 — coordinator prompt 只说 "decompose"，不要求综合 |
| **持久性** | 跨 session 可恢复（`matchSessionMode`） | 单次执行，不可恢复 |
| **Worker 管理** | 可停止 (TaskStop) + 继续 (SendMessage) | Fire-and-forget，只追踪 PR URL |
| **共享状态** | Scratchpad 目录 | 无（worker 完全隔离） |
| **外部事件** | PR subscription | 无 |

**本质区别**：
- Coordinator Mode 是一个 **运行时角色系统** — 改变了 Claude 是谁（system prompt）、能做什么（工具限制）、如何工作（结构化 phases）
- /batch 是一个 **一次性编排指令** — 不改变 Claude 的身份，只给它一个复杂的任务描述

Coordinator Mode 更像一个 OS-level process model；/batch 更像一个 shell script。

---

## 4. AE Plugin 可以学到什么

### 直接可采纳的模式

**a) Synthesis mandate（已适配）**

AE 的 TL 有完整工具访问权限，可以直接读文件验证 worker 发现。这使得 AE 的 synthesis 比 coordinator mode 的更有根据。AE 应采纳 "never delegate understanding" 原则，但以更强的形式：**TL 不仅要综合 worker 文本报告，还应验证关键声明**（读文件确认路径/行号）。

**b) Context overlap heuristic**

Continue vs spawn 的决策表（coordinatorMode.ts Section 5）直接适用于 AE 的 agent 复用决策：
- 研究 agent 刚探索了需要编辑的文件 → 继续该 agent 做实现
- 验证 → 必须新建（新鲜视角，避免实现偏见）
- 错误修复 → 继续（保留错误上下文）
- 完全错误的方向 → 新建（避免上下文锚定）

**c) Fresh-eyes verification**

Coordinator mode 明确要求验证由新 worker 完成。AE 的 `/ae:review` 已遵循此模式（QA agent 独立于实现 agent），但可以在 `/ae:work` 中更显式地强制：实现和验证不能是同一个 agent。

**d) Scratchpad 模式**

跨 agent 知识共享通过文件系统。AE 可以考虑 `.ae/scratch/` 目录，让 agent 在 phases 间积累结构化发现。当前 AE 使用 task list 做协调，但 task 描述不适合存储详细的研究结果。

**e) Worker stopping / redirect (TaskStop)**

AE 目前没有等价机制。如果用户在 `/ae:work` 执行中改变需求，无法中途重定向正在运行的 agent。值得关注。

### 不应采纳的

**a) 工具限制**

AE TL 是讨论调解者，不是任务编排者。限制工具会削弱 TL 验证能力。Coordinator mode 的工具限制是为了强制委派，但 AE TL 需要读取文件来做出有根据的决策。

**b) 完全替换 system prompt**

Coordinator mode 完全替换 system prompt。AE 作为 plugin 不能也不应该替换 CC 的 system prompt — AE 通过 skill prompt 注入工作。

**c) 单 coordinator 瓶颈**

Coordinator mode 所有信息必须流经一个 coordinator 的 context window。AE 的 Agent Teams 设计允许 agent 之间直接通信（SendMessage peer-to-peer），避免了这个瓶颈。不应退化为单点瓶颈。

---

## 5. 设计缺陷与风险（Challenger 总结）

| # | 问题 | 严重度 | 评估 |
|---|------|--------|------|
| 1 | Synthesis without file access = 改写不是理解 | 高 | 成立。Coordinator 无法验证 worker 的文件路径声明。减少了 cascading hallucination 但不能消除。 |
| 2 | 成本模型缺失 | 中 | 成立。"并行是你的超能力" 无成本制约。 |
| 3 | Session resume 竞态条件 | 低-中 | 成立但实际风险低（单进程假设通常成立）。 |
| 4 | 停止的 worker 状态未定义 | 中 | 成立。mid-operation stop 后继续可能进入未知状态。 |
| 5 | PR subscription 只是通知中继 | 低 | 成立。Coordinator 无法看 diff，只能转发事件给 worker。 |
| 6 | 两系统共存缺乏设计 rationale | 中 | 成立。没有文档解释为什么需要两个编排系统。 |
| 7 | Scratchpad gate 可中途关闭 | 低 | 成立但极端 edge case。 |
| 8 | Continue/spawn 无反馈机制 | 低-中 | 成立。没有 worker context 大小信号来辅助决策。 |

---

## 6. 行业对标

| 模式 | LangGraph | CrewAI | AutoGen | Swarm | CC Coordinator |
|------|-----------|--------|---------|-------|----------------|
| 图拓扑 | 预定义 | 预定义 | N/A | 顺序 | 运行时 (prompt) |
| Worker 角色 | Graph nodes | 角色人格 | Named agents | Typed agents | Generic workers |
| 并行 | Graph 分支 | 默认串行 | GroupChat 轮转 | 串行 | **原生异步** |
| Synthesis mandate | 无 | 无 | 无 | 无 | **有（独创）** |
| Continue vs spawn | 无 | 无 | 无 | 无 | **有（独创）** |
| 共享状态 | Typed dict | Vector memory | Full history | 无 | 文件系统 scratchpad |
| Worker 停止 | 无 | 无 | 无 | 无 | 有 (TaskStop) |
| 外部事件订阅 | 无 | 无 | 无 | 无 | 有 (PR events) |

---

## Summary

Coordinator Mode 是 Claude Code 的**运行时编排角色系统**：通过替换 system prompt + 限制工具集，将 Claude 变成一个只能指挥不能动手的 coordinator。它和 Agent Teams 在执行层共享基础设施（AgentTool, SendMessageTool），但在协议层完全独立（不同的激活方式、通信机制、worker 生命周期）。

与 /batch 的 prompt-only 编排相比，coordinator mode 是更深层的角色系统改造：它改变了 Claude 的身份和能力，而非只给它一个编排任务。

**最独创的贡献**："never delegate understanding" synthesis mandate 和 context overlap heuristic — 在主流 AI agent 框架中均无等价物。

**AE 应采纳**：synthesis mandate（以更强的验证形式）、context overlap heuristic、fresh-eyes verification、scratchpad 模式。**不应采纳**：工具限制（AE TL 需要文件访问权）、单 coordinator 瓶颈。

## Possible Next Steps

- `/ae:discuss 011-coordinator-mode-analysis` — 讨论 AE 如何具体编码 synthesis mandate 和 context overlap heuristic
- 对比分析：coordinator mode + Agent Teams + /batch 三者统一视角，规划 AE 的 multi-agent strategy
