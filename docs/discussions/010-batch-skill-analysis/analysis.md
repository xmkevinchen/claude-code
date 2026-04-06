---
id: "010"
title: "Analysis: /batch Skill Deep Dive"
type: analysis
created: 2026-04-05
tags: [batch, parallel-agents, worktree, orchestration, ae-plugin]
---

# Analysis: /batch Skill Deep Dive

## Question

深挖 `/batch` 的内部实现原理，具体能做什么程度的批量操作，以及：1）用户如何正确使用 2）AE plugin 可从中吸取或封装的东西。

---

## 1. 内部实现原理

### 核心文件

`src/skills/bundled/batch.ts` — 仅 124 行。本质是一个 **prompt engineering skill**：不包含任何运行时逻辑，全靠生成结构化 prompt 指导 coordinator LLM 完成三阶段编排。

### 注册链路

```
initBundledSkills() → registerBatchSkill() → registerBundledSkill()
  → 封装为 Command { type: 'prompt', source: 'bundled' }
  → loadAllCommands() → getCommands() 暴露给 slash command 系统
```

用户输入 `/batch <instruction>` → `processSlashCommand()` → `case 'prompt'` → 调用 `getPromptForCommand(args)` → 返回 `buildPrompt(instruction)` → 注入为 user message → 立即触发 model query。

**关键属性：**
- `disableModelInvocation: true` — 模型不能通过 SkillTool 自行调用 /batch，只能用户手动触发
- `userInvocable: true` — 用户可通过 `/batch` 直接调用
- 前置检查：空参数返回使用说明，非 git 仓库返回错误

### 三阶段执行流

**Phase 1 — Research & Plan (Plan Mode)**
1. Coordinator 调用 `EnterPlanModeTool` 进入 plan mode（只读，禁止写文件）
2. 启动前台 research subagent 深入分析代码库
3. 将工作分解为 5-30 个独立单元，每个指定文件列表和变更描述
4. 确定 e2e 测试方案（找不到时用 AskUserQuestionTool 询问用户）
5. 调用 `ExitPlanModeTool` 展示计划 → **用户审批门控**（UI 对话框）
6. 审批通过后返回计划文本给 model

**Phase 2 — Spawn Workers**

审批后，coordinator 在**单条消息**中启动所有 worker agent：
- 每个 worker: `AgentTool({ isolation: "worktree", run_in_background: true })`
- 所有 worker 并行执行

**Worktree 隔离机制（`src/utils/worktree.ts`）：**
```
createAgentWorktree(slug) →
  findCanonicalGitRoot() → 定位主仓库根目录
  getOrCreateWorktree(gitRoot, slug) →
    git worktree add -B worktree-agent-XXXXXXXX
      .claude/worktrees/agent-XXXXXXXX
      origin/<defaultBranch>
  performPostCreationSetup() →
    复制 settings.local.json
    设置 core.hooksPath 共享 hooks
```

Worker 全部操作在 `wrapWithCwd(worktreePath)` 下执行 — cwd 被覆盖为 worktree 路径。

**每个 Worker 的执行管线：**
1. 实现变更
2. 调用 `SkillTool({ skill: "simplify" })` — simplify 本身启动 **3 个并行 review sub-agent**（代码复用/质量/效率审查）
3. 运行测试（unit + e2e）
4. `git commit` + `git push`
5. `gh pr create`
6. 报告 `PR: <url>`

**Phase 3 — Track Progress**

Worker 完成后通过 `enqueueAgentNotification()` 将结果注入 coordinator 的消息队列（非轮询）。通知包含 XML 结构：task-id、status、result、usage、worktree 信息。Coordinator 解析 `PR: <url>` 并更新状态表。

**Worktree 清理（`AgentTool.tsx:644-685`）：**
- 无变更的 worktree: `git worktree remove --force` 自动清理
- 有变更的 worktree（即所有成功 worker）: **保留**，返回路径和分支名

### 资源消耗模型

| 规模 | Worker 数 | simplify sub-agent | 总 agent 数（最少） |
|------|-----------|-------------------|---------------------|
| 最小 | 5 | 5 × 3 = 15 | ~21（含 coordinator + research） |
| 最大 | 30 | 30 × 3 = 90 | ~122+ |

加上每个 worker 可能的 e2e 测试（启动 dev server、浏览器自动化等），实际资源消耗远超表面数字。

---

## 2. 能做什么程度的批量操作

### 适合的场景

| 场景 | 适合度 | 原因 |
|------|--------|------|
| 统一重命名（symbol rename across modules） | ★★★★★ | 每模块独立，无交叉依赖 |
| 批量添加类型注解 | ★★★★★ | 纯加法操作，无冲突风险 |
| lodash → native 迁移 | ★★★★☆ | 每文件独立，但需统一替换模式 |
| import 路径迁移 | ★★★★☆ | 机械操作，可按目录分片 |
| 批量添加测试 | ★★★★☆ | 每模块独立生成 |
| 批量添加 logging/tracing | ★★★★☆ | 加法操作 |
| React → Vue 迁移 | ★★★☆☆ | 可行但需仔细分解，共享类型/状态可能跨单元 |
| 共享 API/类型签名变更 | ★★☆☆☆ | 修改接口 → 所有使用方需同步，破坏独立性假设 |
| 架构级重构 | ★☆☆☆☆ | 隐含执行顺序，不可并行 |

### 硬性约束

- **必须是 git 仓库**（需要 worktree）
- **需要 `gh` CLI 并已认证**（创建 PR）— 但未在启动时验证
- **单元必须真正独立** — 仅靠 prompt 指导，无运行时强制
- **MIN_AGENTS=5, MAX_AGENTS=30** — 仅为 prompt 建议，非硬限制
- **MAX_TOOL_RESULTS_PER_MESSAGE_CHARS=200K** — 大批量时 agent 结果溢出到磁盘

---

## 3. 用户正确使用指南

### Do

1. **指令要具体** — "将所有 lodash.get 替换为 optional chaining，覆盖 src/services/ 下所有文件" 比 "重构代码" 好 100 倍
2. **仔细审查 Phase 1 计划** — 这是你唯一的审批门控。确认：
   - 单元分解是否真正独立（文件列表无重叠）
   - 每个单元描述是否准确
   - e2e 测试方案是否可行
3. **优先选择加法/机械类操作** — 添加类型注解、添加测试、统一 import 格式
4. **确保 `gh` 已认证** — 运行 `gh auth status` 确认，否则 30 个 worker 全部在 PR 创建步骤失败
5. **预估规模** — 如果你的变更只涉及 2-3 个文件，不要用 /batch，直接手写

### Don't

1. **不要用于有执行顺序依赖的变更** — "先提取公共库，再让各模块使用" 不可并行
2. **不要用于修改共享接口/类型** — 接口变更 + 使用方适配必须串行
3. **不要假设 PR 能自动 merge** — /batch 只创建 PR，不处理合并冲突或合并顺序
4. **不要忽略 worktree 清理** — 完成后检查 `.claude/worktrees/`，手动清理残留
5. **不要在大型 monorepo 上盲目使用** — 30 个 agent 的 token 消耗 + simplify 的 3× 放大效应，成本非线性增长

### 完成后的手工步骤

/batch 交付的是 **N 个 PR**，不是合并后的代码。用户还需要：
1. Review 每个 PR
2. 决定合并顺序（dependency-aware）
3. 处理合并冲突（后续 PR 基于同一 base commit，先合并的会导致后续 PR conflict）
4. 可选：创建 integration branch 作为合并目标，而非直接 merge to main
5. 运行完整集成测试

---

## 4. 设计缺陷与风险（Challenger 总结）

### 结构性问题

| # | 问题 | 严重度 | 根因 |
|---|------|--------|------|
| 1 | 独立性假设仅靠 prompt 强制 | 高 | 无运行时依赖检测 |
| 2 | 无 integration branch — PR 直接 target main | 高 | 合并第 2 个 PR 起就可能 conflict |
| 3 | 无 post-merge 集成测试 | 高 | worker 测试在隔离环境，不覆盖交叉影响 |
| 4 | 成本不透明 | 中 | 计划审批不含成本估算，simplify 3× 放大 |
| 5 | MIN_AGENTS=5 强制最小值 | 中 | 2-3 单元的任务被人为拆分 |
| 6 | `gh` 认证未预检 | 中 | 30 个 worker 全部做完实现后统一在 PR 步骤失败 |
| 7 | Worktree 清理不完整 | 低-中 | 成功 worker 的 worktree 全部保留 |
| 8 | Coordinator context window 存活风险 | 低-中 | 30 个长时间异步 agent，coordinator 可能超时 |
| 9 | 用户指令歧义放大 | 中 | 模糊指令 → 30 个 worker 实现错误解读 |

### Challenger 独到发现

- **无人挑战指令歧义问题**：AskUserQuestionTool 仅用于 e2e 方案选择，不用于确认用户意图/范围解读
- **"industry standard" ≠ correct**：所有主流工具采用 worktree-per-agent 不代表 merge conflict 问题被解决，只代表问题被普遍继承
- **Rosie 类比是 category error**：Google 的大规模变更有专用基础设施（LSC tooling、TAP CI），用 Rosie 验证一个 124 行 prompt skill 是错误对标

---

## 5. AE Plugin 可吸取/封装的

### 已验证的好模式（直接吸取）

1. **Prompt-as-orchestrator 模式** — 用结构化 prompt 驱动 coordinator，而非写自定义运行时。轻量、灵活、可迭代
2. **Plan mode 门控** — 强制先计划后执行，用户审批作为唯一门控点。AE 的 discuss→plan→work 流程可参考
3. **Worktree 隔离** — `isolation: "worktree"` 是 Claude Code 原生支持的成熟机制，AE 的并行 agent 也应使用
4. **Simplify 作为质量门** — Worker 完成实现后自动调用 review skill。AE 的 `/ae:work` 已有 QA agent，模式一致
5. **Status table 追踪** — Markdown 表格作为实时进度展示，简单有效

### 高价值封装方向（AE 可做的增强）

**Priority 1 — Pre-spawn 依赖分析**
- 在 Phase 1 后、Phase 2 前，插入一个 dependency analysis agent
- 对每个 work unit 的文件列表做交叉检查：任何文件出现在 2+ 个 unit 中 → 报警
- 用 AST 级别的 symbol reference 检查跨单元依赖（type/interface 被多个 unit 引用）
- 输出 DAG → 用户确认后再 spawn

**Priority 2 — Integration branch 策略**
- Worker PR 不 target main，target `batch/<slug>` integration branch
- 所有 PR merge 到 integration branch 后，运行完整测试
- 通过后创建 "Merge to main" 的最终 PR
- 这把 30 个 PR 的合并冲突问题从 "用户手工解决" 变成 "受控流程"

**Priority 3 — 成本估算 + 确认**
- Phase 1 计划审批时增加成本维度："将启动 N 个 worker（每个含 3 个 simplify sub-agent），预计 token 消耗 ~X"
- 提供 `--dry-run` 模式：只输出计划和成本估算，不 spawn

**Priority 4 — Post-merge 验证 agent**
- Integration branch 上 merge 完所有 PR 后，spawn 专门的 validation agent
- 运行 build + lint + full test suite
- 失败时 attribution：将 CI 失败 map 回具体 PR/unit

**Priority 5 — `gh` 预检 + 清理**
- Phase 1 开始时 `gh auth status` 检查
- 完成后提供清理命令：列出所有 batch 创建的 worktree 和 branch，一键清理

**Priority 6 — Wave 模式**
- 超过 MAX_AGENTS 的任务分 wave 执行：wave 1 (10 units) → merge → wave 2 (10 units)
- 提供 wave 间的 checkpoint 和早期信号反馈

### 不该做的

- **不要重写 /batch** — 它是 prompt-only skill，AE wrapper 应在外层增强，不改内部
- **不要自动重试失败 worker** — AI 重试可能 compound error
- **不要重新解读用户分解** — 用户审批的计划就是最终计划

---

## 6. 与行业对标

| 维度 | /batch | 行业标准 | 差距 |
|------|--------|---------|------|
| Worktree 隔离 | ✅ | ✅ | 无 |
| PR-per-unit | ✅ | ✅ | 无 |
| 模块级分解 | ✅ | ✅ + 依赖图 | 缺依赖分析 |
| 计划审批 | ✅ | ✅ | 缺成本维度 |
| Worker 测试 | ✅ (unit + e2e) | ✅ | 无 |
| 集成测试 | ❌ | ✅ (merge queue) | **缺失** |
| Integration branch | ❌ | ✅ (OpenHands) | **缺失** |
| 冲突检测 | ❌ | 部分 (git merge-tree) | **缺失** |
| Post-merge 门控 | ❌ | ✅ | **缺失** |
| 成本透明 | ❌ | 部分 | **缺失** |
| 清理自动化 | 部分 | ✅ | 有变更的 worktree 不清理 |

---

## Summary

`/batch` 是一个设计精巧的 **prompt-only orchestration skill**（124 行），利用 Claude Code 原生的 worktree 隔离、plan mode 门控、background agent 和 notification queue 实现了 5-30 个并行 worker 的大规模代码变更能力。核心执行链路成熟可靠。

**能力边界**：适合统一、机械、加法类变更（rename、migrate import、add annotations）。不适合有执行顺序依赖或修改共享接口的变更。

**主要缺陷**：独立性假设无运行时强制；无 integration branch 导致合并是用户的手工活；无 post-merge 集成测试；成本不透明（simplify 3× 放大效应）。

**AE plugin 机会**：最高价值是在 /batch 外层包装 pre-spawn 依赖分析 + integration branch 策略 + post-merge 验证，把 /batch 从 "创建 N 个 PR" 升级为 "安全地交付一个经过验证的大规模变更"。

## Possible Next Steps

- `/ae:discuss 010-batch-skill-analysis` — 讨论 AE wrapper 的具体设计决策（integration branch 策略、依赖分析深度、wave 模式触发条件）
- `/ae:plan` — 如果方向明确，直接规划 AE batch wrapper 的实现
