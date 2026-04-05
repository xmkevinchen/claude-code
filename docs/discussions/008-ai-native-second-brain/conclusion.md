---
id: "008"
title: "AI-native Second Brain MVP Phase 1 — Conclusion"
concluded: 2026-04-05
plan: ""
---

# AI-native Second Brain MVP Phase 1 — Conclusion

4 agents（architect + archaeologist + challenger + codex-proxy），2 轮讨论后的收敛结论。

---

## MVP Phase 1 方案

### 交付形态：MCP Server（stdio）

注册为 Claude Code 的 local stdio MCP server。用户在 `~/.claude/settings.json` 的 `mcpServers` 里加一行配置即可。Claude Code 按需启动，AE 所有 subagent 自动继承——零额外配置。

代码验证：`src/services/mcp/types.ts:28-35` 支持 stdio server，`runAgent.ts:88-166` 确认 subagent 继承 parent MCP tools。

### 3 个 MCP 工具

| 工具 | 功能 | Phase 1 实现 |
|------|------|-------------|
| `memory_search(query, scope?)` | 搜索记忆，返回结果 + provenance | SQLite FTS5 + 向量相似度混合搜索 |
| `memory_ingest(entry)` | 摄入一条记忆 | 解析 YAML frontmatter + section 提取 |
| `memory_invalidate(entry_id, reason)` | 标记记忆失效 | 设置 `valid_until` + `superseded_by` |

Phase 2 增加：`memory_cite(entry_id, context)`（引用信号，强化 Dreaming 评分）、`list_memories(filter)`。

### 摄入端：AE output watcher only

Phase 1 只做 AE pipeline 产出的摄入。不做 Claude Code session hook（推到 Phase 2）。

**Watch 的文件**：
- `docs/discussions/**/conclusion.md` — 决策记忆（最高价值）
- `docs/reviews/**/*.md` — 经验教训
- `docs/plans/**/*.md` — 方案记忆
- `docs/analyses/**/*retrospect*.md` — 趋势记忆

**不 watch 的**：round-NN.md（中间过程）、analysis.md（被 conclusion 替代）、topic summary（被 conclusion 替代）。

**实现方式**：文件系统 watcher（fsnotify / chokidar），detect file create/modify → 解析 frontmatter + 正文 → 调用 `memory_ingest`。

### 反哺端：ae:analyze SKILL.md 修改（post-research 注入）

在 ae:analyze 的研究阶段完成后、synthesis 开始前，加入 Second Brain 查询：

```
### 3.5 Prior Context (from Second Brain)

Before synthesizing, search Second Brain for prior decisions on this topic:
- Call memory_search with the research topic
- If results found, present as "Round 0: Prior Decisions" with provenance
- Agents synthesize with explicit awareness of prior decisions
- Note whether current evidence confirms, updates, or contradicts prior decisions
```

这是 ae:analyze SKILL.md 的 ~3 行修改。Post-research 位置避免 anchoring bias（Architect 提出，全员接受）。

**为什么不用 Claude Code SessionStart hook 替代**：SessionStart 在 session 级别触发，不在 ae:analyze 级别。注入时机不对，且 AE subagent 看不到主 session 的 hook 输出。

**Phase 1 只改 ae:analyze**。ae:discuss 和 ae:plan 的反哋推到 Phase 2，原因：
- ae:discuss 的注入需要按 agent 角色差异化（archaeologist 看历史证据，challenger 看历史挑战），设计更复杂
- ae:plan 依赖 ae:discuss 的输出，隐式获得了 Second Brain 的信号

### 存储：全局 SQLite，per-project 默认搜索

**存储位置**：`~/.second-brain/db.sqlite`（全局，不在任何项目内）

**每条记忆的 schema**：
```
id: uuid
project_id: text       (git-inferred: remote URL 或 local path hash)
source_file: text      (原始文件路径)
source_type: text      (conclusion | review | plan | retrospect)
knowledge_type: text   (decisional | experiential | factual)
title: text
content: text
entities: text[]       (从 frontmatter tags + Decision Summary 提取)
valid_from: timestamp
valid_until: timestamp? (null = 当前有效)
superseded_by: uuid?
recall_count: int
avg_relevance: float
last_recalled: timestamp
embedding: blob        (向量)
created_at: timestamp
```

**project_id 推断**：git remote URL（`git remote get-url origin`），monorepo 用 `.ae/` 相对路径做 sub-identifier。不需要 AE frontmatter 改动。

**搜索默认 scope = 当前项目**。`scope: "global"` 参数可选但 Phase 1 默认关闭。Dreaming 在全局运行（跨项目评分对信号质量有帮助）。

### 简化 Dreaming（Phase 1）

只跟踪两个信号：recall_count（被搜索次数）+ avg_relevance（平均匹配得分）。

**提升规则**：
- `recall_count >= 3` AND `avg_relevance >= 0.65`，14 天窗口内
- 达标 → 标记为 long-term（搜索时权重提升）
- 每日运行（cron 或 MCP server 内置 timer）

**不做的**（推到 Phase 2）：
- consolidation / conceptual signals（需要读 MEMORY.md 和内容分析）
- cite() 信号（需要 AE 额外调用 memory_cite）
- 衰减/归档（Phase 1 记忆只增不减，靠 valid_until 手动失效）

**实现量**：~150 行，无 OpenClaw 依赖。

### 矛盾检测（MVP 最小版）

**Entity-tag 定向比对 + Temporal Validity**：

摄入时提取 entities（从 frontmatter tags + Decision Summary 表的 Topic 列）。新记忆摄入时，只跟拥有重叠 entity 的已有记忆做语义比对。

**处理逻辑**：
- entity 重叠 + 语义相似 + `knowledge_type == decisional` → 标"演化候选"：提示"新决策可能替代旧决策，是否将旧的标为失效？"
- entity 重叠 + 语义相反 + 时间间隔 <30 天 → 标"冲突"
- 用户确认 → 设置旧记忆 `valid_until` + `superseded_by`

**不做的**：跨术语语义冲突检测（"Use Redis" vs "Store in PostgreSQL"）——需要 embedding 对比，推 Phase 2 的关联发现层。

### 冷启动：批量导入

已有 AE discussions 的用户：`second-brain import --dir .ae/discussions/` 批量导入所有 conclusion.md + review.md。

**导入策略**：
- 直接进入长期记忆（不走短期等待 Dreaming），因为 AE 产出已经过多 agent 讨论验证
- 标记 `recall_count = 0`（未被实际引用验证）
- 第一次被 ae:analyze 引用后进入正常 Dreaming 循环

**不用 AI 判断做冷启动**——避免 Challenger 的场景 C（冷启动期 AI 判断错误被放大）。

### 可观测性（MVP metrics）

| Metric | 怎么收集 | 判断标准 |
|--------|---------|---------|
| `context_injection_rate` | memory_search 返回非空的比例 | > 60% = 闭环在 work |
| `stale_citation_rate` | 被引用记忆中 `valid_until` 已过期的比例 | < 10% = 知识在更新 |
| `conflict_detection_rate` | 新摄入中触发冲突/演化标记的比例 | 1-15% = 检测灵敏度正常 |
| `memory_age_at_retrieval` | 被引用记忆的平均年龄 | 持续走高且无更新 = 记忆老化 |

**Holdout 实验**（Phase 1 简化版）：MCP server 端基于 session hash 对 20% session 返回空结果。对比 holdout vs injection session 的 ae:review Outcome Statistics。但这是 Phase 1 后期做的分析，不是 Day 1 功能。

**自然准实验**（Day 1 可做）：对比有 context injection 的 run 和没有（Second Brain 空或搜索未命中）的 run，不需要主动关闭 injection。

---

## 各 Topic 决策摘要

### Topic 1: MVP Scope

| 组件 | Phase 1 | Phase 2 |
|------|---------|---------|
| MCP server（3 工具） | ✅ | 加 cite + list |
| AE output watcher | ✅（4 类文件） | 加 GitHub webhook, Slack bot |
| Claude Code session hook | ❌ | ✅（SessionEnd + SessionStart） |
| 简化 Dreaming | ✅（frequency + relevance） | 完整 6 信号 + cite |
| ae:analyze 反哺 | ✅（post-research Round 0） | 加 ae:discuss 按角色差异化注入 |
| 矛盾检测 | ✅（entity-tag + temporal validity） | 加跨术语语义检测 |
| 衰减/归档 | ❌ | ✅ |
| 关联发现 | ❌ | ✅ |
| 团队级 | ❌ | Phase 3 |

### Topic 2: 跨项目

**决策**：全局存储，per-project 默认搜索。
- project_id 从 git remote URL 推断，不需要 AE 改动
- 搜索默认 scope = 当前项目
- Dreaming 全局运行
- 跨项目搜索作为可选参数预留，Phase 1 不开放

**理由**：全局存储的实现成本只比 per-project 多一个 `project_id` 列和一个搜索参数。但 per-project-first 如果后期迁移到全局会破坏 Dreaming 历史。

### Topic 3: AE Gaps

**需要改的**：
- ae:analyze SKILL.md：+3 行（post-research memory_search + Round 0 展示）
- AE conclusion.md 模板：+1 行 frontmatter（`entities: [...]`）

**不需要改的**：
- AE pipeline.yml
- AE 其他 skill 的 SKILL.md
- AE agent 定义
- AE 不需要知道 Second Brain 存在——MCP tool 注册在 Claude Code settings 里，AE subagent 自动继承

### Topic 4: 负面螺旋防护

**Phase 1 防线**：
1. Entity-tag + temporal validity 矛盾检测（摄入时）
2. 反哋注入必须非静默（Round 0 带 provenance 展示）
3. 批量导入不用 AI 判断（避免冷启动期错误放大）
4. 4 个可观测性 metrics + 自然准实验

**识别到但推迟的风险**：
- Challenger 的场景 B（workaround 被固化为最佳实践）：需要依赖库版本变更触发扫描——Phase 2
- ae:discuss 注入时所有 agents 拿到相同上下文降低讨论质量——Phase 2 做按角色差异化注入

---

## 实现估算

| 组件 | 工作量 | 依赖 |
|------|--------|------|
| MCP server 框架（stdio） | 2-3 天 | 无 |
| SQLite + hybrid search | 5-7 天 | 无（重写，不用 OpenClaw SDK） |
| AE file watcher | 2-3 天 | MCP server |
| 简化 Dreaming | 2-3 天 | SQLite |
| 矛盾检测 | 3-4 天 | SQLite + search |
| ae:analyze SKILL.md 修改 | 0.5 天 | MCP server |
| 批量导入 CLI | 1-2 天 | SQLite + watcher |
| Metrics dashboard | 1-2 天 | SQLite |
| **总计** | **~3 周** | |

---

## Doodlestein Review

未运行 Doodlestein（Investigation Mode，TL 判断不需要——讨论产生的是具体实现方案而非重大架构决策，Architect 和 Challenger 的对抗已经充分覆盖了盲点）。

## Team Composition

| Agent | Role | Backend | Focus |
|-------|------|---------|-------|
| TL | Moderator | Claude | Synthesis + tie-breaking |
| architect | 架构设计 | Claude | MVP 技术方案 |
| archaeologist | 代码验证 | Claude | 可行性确认 |
| challenger | 风险分析 | Claude | 负面螺旋 + scope 挑战 |
| codex-proxy | 跨家族 | Codex | 竞品对比 + metrics |

## Process Metadata
- Discussion rounds: 2 (Architect did 3, others 2)
- Topics: 4 (4 converged)
- Autonomous decisions: 4
- User escalations: 0

## Next Steps
→ `/ae:plan` — 制定实施计划（基于此 conclusion 的 3 周估算）
→ 验证闭环的最短路径：MCP server + AE watcher + ae:analyze 修改 + 批量导入
