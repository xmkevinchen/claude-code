---
id: "008"
title: "AI-native Second Brain — 产品构想"
type: product-vision
created: 2026-04-05
status: draft
origin: "Claude Code 源码分析 (007) + OpenClaw 记忆系统对比 + AE pipeline 输出分析"
tags: [product, second-brain, memory, knowledge-management]
---

# AI-native Second Brain

## 一句话

AI 是写入者和维护者，人是消费者和裁判。

传统 Second Brain（Obsidian / Logseq / Notion）的问题：全靠人写、人整理、人连接。AI-native 的核心翻转：信息从各个工具被动流入，在使用中自动筛选和组织，人只需要在关键时刻做判断。

---

## 从哪里来的

### Claude Code 的四套记忆机制

Claude Code 为了让 AI 编程助手不失忆，造了四套独立系统：

- **Compaction**（对话压缩）：对话快满了就自动压缩。三档——删旧 tool result、用摘要替代已总结消息、全量压缩。跟 Session Memory 有集成（先用 SM 的提取结果做压缩，不够再全量）。
- **Session Memory**（对话内提取）：后台 fork 一个 subagent，按固定模板（任务状态、文件列表、错误经验、关键结果）提取结构化信息。每 5000 token + 3 次工具调用更新一次。
- **Scratchpad**（多 agent 共享目录）：`/tmp` 下的临时目录，coordinator 模式下 worker agents 可以自由读写。Session 结束即消亡。
- **AutoDream**（跨会话整合）：24 小时 + 5 次对话后触发。扫原始对话日志，整理成持久记忆文件。

问题：AutoDream 不读 Session Memory 的结构化提取结果（从头 grep 日志），Scratchpad 知识随 session 消亡，没有"什么值得记"的量化判断。

### OpenClaw 做对的一步

OpenClaw 的 Dreaming 系统跟踪每一次 `memory_search` 调用，用加权评分（频率 35% + 相关度 35% + 多样性 15% + 新鲜度 15%）决定要不要从短期记忆提升到长期记忆。

**关键区别**：不是让 AI 判断"这个重要不重要"，而是让使用行为说话。被反复调用的信息就是重要的，从不被调用的自然消失。

OpenClaw 还做了一件正确的事：把上下文管理抽象成接口（Context Engine），记忆后端可替换。

### AE pipeline——天然的结构化知识生产线

AE plugin 的 analyze → discuss → plan → work → review → retrospect 流程，每一步都在产出结构化的 Markdown 文件：

| AE 产出 | 包含什么 | 为什么有价值 |
|---------|---------|------------|
| conclusion.md | 决策 + 理由 + 证据 + 可逆性评估 + Doodlestein 挑战记录 | 不只是"选了方案 A"，而是"为什么选 A、考虑了什么、谁反对了什么" |
| plan.md | 步骤 + 预期文件 + 验收标准 | 实现方案的完整契约，带漂移检测基线 |
| review.md | P1/P2/P3 发现 + 修复记录 + Outcome Statistics | "什么场景容易出什么问题" |
| retrospect.md | 趋势分析（rework rate / P1 escape / drift / auto-pass） | "团队在变好还是变差，为什么" |
| backlog/*.md | 延后工作 + 原因 + 优先级 | "什么被推迟了，为什么，什么时候该回来处理" |

这些产出**天生结构化**——有 YAML frontmatter、固定 schema、类型标注。不需要 AI 再做提取，直接摄入就行。

更重要的是：AE 产出的是**决策上下文**，恰好是传统 Second Brain 最缺的东西。Obsidian 里你能记"我们选了 React"，但很难保留"为什么选 React、当时考虑了什么、谁反对了什么、后来验证结果怎么样"。AE 的输出天然包含这些。

### 还差什么

三个项目（Claude Code / OpenClaw / AE）各自做了一部分，但没有人把它们连起来：
1. Claude Code 和 OpenClaw 只有跟 AI 聊天的信息能进入记忆。AE pipeline 的结构化产出、GitHub 上的 review、Slack 里的讨论——不会自动流入。
2. 记忆之间没有关联。A 和 B 说的是相关的事，但搜到 A 不会顺带给你 B。
3. 只有个人级。没有"团队共享记忆"的概念。
4. **没有闭环**：AE 产出知识 → 但下次 AE 运行时不会自动查阅之前的知识。每次都是重头开始。

---

## 核心设计：螺旋上升的闭环

这个产品的核心不是四层架构——而是一个闭环：

```
AI 工作流（Claude Code / AE pipeline / OpenClaw）
    │
    │ 产出知识（对话经验、决策记录、review 发现、趋势分析）
    ↓
Second Brain 被动摄入
    │
    │ 行为驱动筛选（Dreaming: 频率 × 相关度 × 多样性 × 新鲜度）
    ↓
结构化长期记忆（个人 + 团队）
    │
    │ 自动关联（语义 + 共现 + 时间线演化）
    ↓
下次 AI 工作流的输入上下文
    │
    │ "上次在类似场景下的决策是什么？结果怎么样？"
    ↓
更好的 AI 输出 → 更高质量的知识产出 → 更丰富的 Second Brain → ...
```

**不是线性的"输入→处理→输出"，是螺旋上升**：
- 第一次 `ae:analyze` 一个问题，从零开始研究
- 分析结果进入 Second Brain
- 第二次遇到类似问题，`ae:analyze` 自动从 Second Brain 拉取上次的决策和经验作为输入上下文
- 这次的分析质量更高，因为不用重新发现上次已经知道的东西
- 新的分析结果又反哺 Second Brain，纠正或补充之前的知识

每转一圈，知识积累更深，AI 工作的起点更高。

### 具体怎么闭环

**输出端（AI 工作流 → Second Brain）**：

| 来源 | 摄入方式 | 摄入内容 | 结构化程度 |
|------|---------|---------|----------|
| Claude Code | Session 结束时 hook | 改了什么、遇到什么 error、怎么解的 | 中（Session Memory 模板） |
| AE pipeline | 文件 watcher 或 post-write hook | 决策 + 理由 + 证据 + 挑战 + 验证结果 | **极高**（固定 schema） |
| OpenClaw | 已有的 session-memory hook | 对话摘要、action items | 中 |
| GitHub | Webhook | PR review 观点、issue 讨论 | 低-中 |
| Slack / Discord | Bot | 技术讨论、决策 | 低 |

**输入端（Second Brain → AI 工作流）**：

| 消费者 | 怎么拉取 | 拉取什么 |
|--------|---------|---------|
| `ae:analyze` | 研究开始前自动搜索 Second Brain | 同一项目/同一领域之前的分析和决策 |
| `ae:discuss` | 讨论前加载相关 prior decisions | "上次我们讨论过类似的问题，结论是 X" |
| `ae:plan` | 规划时参考之前的 review findings 和 retrospect 趋势 | "这个模块上次 review 发现 DB migration 容易出 P1" |
| `ae:work` | 实现时参考之前同类任务的经验 | "上次改这个 API 时要注意 timeout 设置" |
| Claude Code 日常对话 | `memory_search` 自动触发 | 相关的过往经验和决策 |

**闭环的关键**：AE pipeline 和 Claude Code 不需要改动核心逻辑。只需要：
1. 在 AI 工作流启动时，自动把 Second Brain 的相关内容注入到 system prompt 或 user context 里
2. 在 AI 工作流结束时，自动把产出的知识摄入 Second Brain

这两个都是 hook——不侵入核心代码。

---

## 产品设计

### 定位

面向使用 AI 工具的技术团队（3-15 人）。不是 Notion 的竞品（那是协作文档工具），是 AI 工作流的"记忆层"——所有 AI 工具产生的知识自动汇聚、自动筛选、自动关联，并且反哺回 AI 工作流。

### 目标用户

- 每天用 Claude Code / Cursor / Copilot 写代码的工程师
- 用 AE pipeline 或类似工具做研究、做设计、做代码审查的技术团队
- 希望"团队里每个人学到的东西，其他人不用重新踩一遍坑"的 tech lead

### 不做什么

- 不是文档工具（不跟 Notion / Confluence 竞争）
- 不是笔记软件（不跟 Obsidian / Logseq 竞争）
- 不是搜索引擎（不跟 Glean / Dashworks 竞争）
- 只做"AI 工作流产出的知识的管理和反哺"这一件事

---

## 四层架构

### Layer 1: 被动摄入

**核心原则**：用户不需要"记得往里写"。信息从各个工具自动流入。

**已有的参考实现**：
- OpenClaw 的 session-memory hook：`/new` 或 `/reset` 时自动保存对话摘要
- Claude Code 的 Session Memory：后台 fork agent 按模板提取
- OpenClaw 的 Context Engine `ingest()` / `ingestBatch()`：接受消息的标准接口

**两类来源**：

**操作记忆**——"今天做了什么"：
- Claude Code session hook：改了什么文件、遇到什么 error、怎么解的
- OpenClaw session-memory hook：对话摘要、action items

**决策记忆**——"为什么这样做"：
- AE pipeline output：conclusion.md（决策+理由）、plan.md（方案+标准）、review.md（发现+趋势）
- GitHub webhook：PR review 中的架构观点
- Slack/Discord bot：技术讨论中的决策

**摄入不等于记住**。所有摄入的信息先进短期存储（daily notes），成本极低。只有经过筛选层验证的才会进入长期记忆。

AE pipeline 的产出因为天生结构化（YAML frontmatter + 固定 schema），可以直接提取为知识条目，不需要 AI 做额外的理解和转化。这是它比其他来源更高效的原因。

### Layer 2: 行为驱动的自然选择

**核心原则**：什么信息在实际工作中被反复调用，什么就值得长期记住。不依赖 AI 的"判断力"。

**已有的参考实现**：
- OpenClaw Dreaming：跟踪 `memory_search` 调用 → 加权评分 → 提升/淘汰
- Claude Code AutoDream：时间+会话数 gate → AI 扫日志整理（较粗糙）

**评分公式**：

```
短期存储（daily notes，所有摄入的信息）
    ↓ 跟踪每一次被搜索/被引用/被 AI 使用的行为
    ↓ 评分：频率 × 相关度 × 多样性 × 新鲜度 × [团队级: 团队普遍性]
    ↓
长期记忆
    ↓ 持续监控：长时间不被使用 → 分数衰减 → 降级回短期或归档
    ↓
归档（不再活跃但还在——搜索时权重降低，不主动呈现）
```

**闭环加速器**：当一条记忆被 AE pipeline 引用（比如 `ae:analyze` 拉取了之前的决策作为输入），这次引用本身就是一次高权重的"被使用"事件。AE 产出质量越高 → 越常被后续 AE 引用 → 在 Dreaming 中分数越高 → 越持久地保存在长期记忆中。**好知识自我强化，差知识自然淘汰**。

**矛盾检测**：新信息跟已有记忆矛盾时自动标记。不自动删除旧的，标注"这两条信息有冲突，你可能需要判断"。标注为冲突的记忆在搜索结果中同时呈现两面，让用户或 AI 做最终判断。

### Layer 3: 自动关联

**核心原则**：知识的价值不只在条目本身，更在条目之间的联系。

**三种关联机制**：

1. **语义相似**：向量相似度超过阈值 → 自动建立弱关联。"这条笔记和那条笔记说的是相关的事"。
2. **共现关联**：A 和 B 经常在同一个对话/工作上下文中被一起搜到 → 建立使用关联。比语义相似更强的信号——两条信息可能字面上不像，但在实践中经常一起用。
3. **时间线演化**：同一个主题的认知随时间变化。"1 月决定用方案 A（discussion 001），3 月 review 发现 A 有性能问题（review 003），5 月改用方案 B（discussion 007）"。不删旧的，标注演化关系。

**对闭环的意义**：当 `ae:analyze` 搜索 Second Brain 时，不只拿到直接匹配的记忆，还拿到关联的记忆。"你问的是 API 超时问题。直接相关：上次设了 30s timeout。时间线关联：更早之前试过 10s，review 发现不够。共现关联：这个 API 跟认证模块经常一起出问题。"

这让 AI 的输入上下文更丰富，产出质量更高，形成更强的螺旋上升。

### Layer 4: 团队级别

**核心原则**：团队中多个人独立学到的同一件事，说明这是组织级别的知识。

**两层存储**：

```
个人记忆（私有，只有我能看）
    ↕ 自动建议提升 / 人工下沉
团队记忆（共享，团队成员都能看）
```

**提升机制**：当团队中 N 个人（默认 3）的个人记忆里有语义相似的条目时，系统自动建议提升到团队记忆。不是自动提升——涉及隐私和准确性，需要人确认。

**团队记忆的特殊来源**：AE pipeline 的 conclusion.md 和 review.md 本身就是团队级别的产出（多 agent 讨论产生的决策、多角度 review 的发现）。这些可以直接进入团队记忆，不需要经过"3 个人都记了类似的"的检测——因为它们本身就代表团队共识。

**团队闭环**：新人入职 → 自动加载团队记忆 → 第一次用 `ae:analyze` 就能获得团队之前积累的决策上下文 → 不需要翻历史或问同事。

---

## 技术基础

### 可以直接复用的

| 来源 | 组件 | 用途 |
|------|------|------|
| OpenClaw | Context Engine 接口 | 摄入层的标准接口 |
| OpenClaw | Dreaming 评分模型 | 筛选层的核心算法 |
| OpenClaw | memory_search（语义+关键词混合） | 搜索层 |
| OpenClaw | 可插拔后端（SQLite/LanceDB） | 存储层 |
| Claude Code | Session Memory 模板 | 操作记忆的结构化提取 |
| Claude Code | Permission 系统设计 | 团队级权限控制的参考 |
| AE plugin | 全套 output schema | 决策记忆的直接来源 |
| AE plugin | pipeline.yml 配置模式 | 输出路径和命名规范 |

### 需要新建的

- 多源摄入适配器（Claude Code hook、AE output watcher、GitHub webhook、Slack bot）
- 关联发现引擎（语义相似 + 共现 + 时间线）
- 团队 Dreaming（跨个人记忆的相似度检测 + 提升机制）
- 矛盾检测和标注
- 衰减和归档机制
- **反哺 hook**：AI 工作流启动时从 Second Brain 拉取相关上下文

### 不需要的

- 知识图谱（过度工程，简单的关联列表就够）
- 自然语言生成摘要（基于评分的筛选比 AI 生成摘要更可靠）
- 复杂的 UI（目标用户是工程师，Markdown + CLI + IDE 插件就够）

---

## 关键风险

### "garbage in" 风险

所有被动摄入系统的天敌：噪音太多。

缓解：
- AE pipeline 产出天生结构化，噪音极低——这是最干净的来源
- 其他来源摄入时做一次轻量 triage
- 宁可多进噪音（被 Dreaming 淘汰），不要漏掉有价值的信息

### "冷启动" 问题

新用户没有使用行为数据，Dreaming 无法评分。

缓解：前两周用 AI 判断（类似 AutoDream），积累足够使用数据后切到行为驱动模式。如果用户已有 AE pipeline 产出（比如已有 20+ 个 discussions），可以直接批量导入——冷启动变成了热启动。

### 隐私

团队级别，个人记忆可能包含私人信息。提升到团队记忆前必须经过人确认。

### 可信度

AI 提取的信息可能不准确。每条记忆标注来源（哪个对话/工具/时间），让用户判断。长时间不被验证的记忆自动降权。

### 闭环的负面螺旋

如果早期摄入了错误知识，后续 AI 工作流引用了它，产出了基于错误前提的新知识，这个错误会自我强化。

缓解：矛盾检测是关键防线。当新信息跟已有记忆冲突时，标记冲突而不是默默覆盖或默默延续。用户/AI 看到冲突标记后做判断。另外，所有记忆的来源溯源让用户能追查到错误的源头。

---

## 最小可行产品

**Phase 1：个人闭环**
- Claude Code session hook（操作记忆）
- AE pipeline output watcher（决策记忆）
- 本地 SQLite + 向量索引
- `memory search` CLI 命令
- Dreaming 评分（复用 OpenClaw 逻辑）
- **反哺 hook**：`ae:analyze` 启动时自动搜索 Second Brain，注入相关上下文

这个 phase 就可以验证闭环是否成立：AE 产出 → 摄入 → 下次 AE 引用 → 更好的产出。

**Phase 2：更多来源 + 关联**
- GitHub webhook + Slack bot
- 摄入时轻量 triage
- 语义相似 + 共现关联

**Phase 3：团队级**
- 跨个人记忆的相似度检测
- AE 团队产出直接进团队记忆
- 提升机制 + 确认流程
- 团队搜索（合并个人库 + 团队库）

---

## Open Questions

1. **存储格式**：Markdown 文件（人类可读、可 git 管理）还是数据库（性能、关联）？OpenClaw 的 Markdown + 向量索引组合可能是最好的折中。

2. **反哺的粒度**：给 AI 多少上下文？太少没用，太多浪费 token。可能需要根据任务类型和 context window 预算动态调整。

3. **商业模式**：个人免费/开源 + 团队级付费？还是全部开源，靠托管服务赚钱？

4. **跟 OpenClaw 的关系**：作为 OpenClaw 插件还是独立产品？底层复用 OpenClaw 组件（Context Engine、Dreaming、memory_search），但产品定位独立。

5. **AE pipeline 产出的去重**：一个 discussion 可能产生 index.md + analysis.md + conclusion.md + 多个 topic + round 文件。全部摄入会有大量重复信息。需要只摄入"最终产出"（conclusion.md、plan.md、review.md）还是全量摄入让 Dreaming 去筛？
