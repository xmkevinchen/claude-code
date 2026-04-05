---
id: "007"
title: "Claude Code 源码深度探索 — Conclusion"
concluded: 2026-04-04
plan: ""
---

# Claude Code 源码里还藏着什么

8 个 agent（4 研究 + 3 Doodlestein 挑战 + 1 跨家族）对 Claude Code 源码做了两轮讨论。以下是经过交叉验证和对抗性挑战后的结论。

---

## 一、Anthropic 在内部测试什么

源码里有 88-89 个 feature flag。大部分最厉害的功能外部用户用不到——它们在公开版本的构建过程中被直接删除（Dead Code Elimination），或者被远程开关（GrowthBook）默认关闭。

### 值得关注的

**KAIROS — "AI 同事"，不是 "AI 工具"**

不是你打开才工作的助手，而是一个后台 daemon。从源码碎片拼出的能力：
- 订阅 GitHub PR 事件，主动通知你
- 早上给你发 briefing（morning-checkin cron task）
- 晚上自己整理记忆（nightly dream skill，凌晨 1-5 点触发）
- 推送通知到系统

核心实现文件（`src/assistant/index.ts`、`gate.ts`）在公开源码中不存在。只有 Anthropic 内部构建版本包含。这是他们的核心方向：**把 AI 从被动工具变成主动代理人**。

**`/batch` — 一句话开 30 个分支**

这个是公开可用的。`MIN_AGENTS = 5, MAX_AGENTS = 30`。一条命令：分析任务 → 拆成 5-30 个子任务 → 每个 agent 在独立 git worktree 里工作 → 自动写代码、跑测试、提交、开 PR → 汇报结果。

不是原型，是生产代码。

**VERIFICATION_AGENT — 专职找茬的 AI**

你写完代码，它来做对抗性验证：跑 build、test、typecheck，然后故意搞并发测试、边界值测试、幂等性测试。有 PASS/FAIL/PARTIAL 三种判定。系统提示里列了七种"AI 自我欺骗"的借口（比如"代码看起来是对的"）并要求自己识别和对抗这些倾向。

但：**双重开关保护，外部用户用不到。** 代码完整，方向已确认，只是还没推给公众。

**BYOC / Self-Hosted Runner — 企业级 AI 基础设施**

让公司在自己的服务器上跑 Claude agents。有注册、心跳 API。这不是"我电脑上有个助手"，这是"公司的 CI/CD 里常驻运行着一批 AI agents"。

是意图信号，不是已成事实。代码在那，但离大规模部署还有距离。

### 不值得过度解读的

- **autoDream**（后台记忆整合）：代码完整但默认关闭，需要 Anthropic 显式推送才激活。说"AI 在你睡觉时学习"太夸张——更准确是"一套设计好的后台任务，等着被打开"。
- **Buddy**（虚拟宠物）：认真做的产品（18 物种、确定性抽卡、ASCII 动画），但本质是使用激励机制，不影响核心能力。
- **ULTRAPLAN**：把规划任务发到 Anthropic 云端跑 Opus。之前说"本地-云端混合"是错的——本地只负责发送和接收，规划全在远端执行。

---

## 二、产品点子

### 最值得做：AI 记忆统一层

Claude Code 里有四套独立的"记忆"系统：

| 什么时候用 | 系统 | 干什么的 |
|-----------|------|---------|
| 对话太长了 | Compaction（11 个文件） | 自动压缩早期对话，释放空间 |
| 一次对话结束后 | Session Memory | 提取关键信息（文件、错误、经验） |
| 多个 AI 一起干活时 | Scratchpad | 共享白板，跨 agent 传递知识 |
| 你几天没用它 | AutoDream | 扫历史对话，整理成持久记忆 |

**问题**：这四套系统互相不知道对方存在。压缩对话时可能丢掉 Session Memory 已经标记的关键信息。AutoDream 整理记忆时不知道 Scratchpad 上有什么重要决策。

**机会**：把这四层统一成一个协议。任何做 AI agent 的公司都需要解决"AI 怎么记住东西"这个问题。现在没有标准方案。这是代码库里已经存在、已经验证、只需要统一的东西——不需要从零开始。

### 值得做：AI 权限系统

Claude Code 的权限系统是 23 个文件，包含：规则引擎 + LLM 分类器混合决策 + bash 命令危险度检测 + 影子规则检测。

所有做 AI agent 的公司都头疼同一个问题：**怎么让 AI 安全地操作真实系统**。这套东西可以抽出来做成独立中间件。类比：Kubernetes 有 OPA 做策略管理，AI agent 领域还没有等价物。

### 值得做：对话无限续命

Context Compaction 有三档压缩（micro/partial/full），支持环境变量覆盖窗口大小，有 analysis + summary 分离的策略。做长对话 AI 产品的团队都需要这个能力，目前没有好用的、模型无关的开源方案。

### 不建议做：提取多 Agent 协调协议

Claude Code 的 agent 协调系统（coordinator prompt + task notification XML + mailbox IPC）设计得不错，但：
1. `runAgent.ts` 前 80 行有 30+ 个内部依赖，提取成本极高
2. MCP（Model Context Protocol）大概率会自己加 agent-to-agent 通信标准
3. 6 个月后你提取的东西可能就被官方标准淘汰了

**Hedge**：每周花 15 分钟跟踪 MCP 规范动态，零代码投入。

### 有价值但小众：Terminal 渲染器

96 个文件，完整的 React reconciler + Yoga flexbox 布局 + ANSI 解析器。技术含量高，但：
- 是基于 Ink 库改的（不是从零写的），IP 状况不清楚
- 只有 3 处与 Claude Code 内部耦合，分离可行但要花精力
- 终端 UI 框架的市场本身很小

---

## 三、对人的冲击

### 自动化到了什么程度

**执行层**：75-85% 的日常编程执行工作已经可以被自动化。`/batch` 一句话并行开 5-30 个 PR，coordinator 模式覆盖 research → synthesis → implementation → verification 全流程。这不是未来——是现在就能跑的代码。

**验证层**：VERIFICATION_AGENT 已经写好，能做对抗性测试并出正式判定。但目前只在 Anthropic 内部测试。执行链是生产级的，验证链是蓝图。

**规划层**：PlanAgent 和 ULTRAPLAN 能做系统级规划。但规划的审批节点（ExitPlanMode）仍然需要人参与——至少在公开版本中是这样。有意思的是，在 swarm 内部，plan 审批已经被自动化了（leader 自动 approve）。

### 谁先受影响

**初级程序员**：1-3 年内岗位需求明显减少。"按规格实现功能、写测试、提 PR、改 review 意见"——`/batch` 的 worker 指令就是对这份工作的完整描述。不是线性消失，是阶跃式压缩：先是需求减少，然后当 AI 对 AI 的验证可信度提高后，连"监督 AI 输出"这个岗位也会被压缩。

**高级程序员**：价值从"写好代码"转向"给 AI 写清晰的规格、判断 AI 输出是否可靠"。但这个价值不是程序员独有的——能写清晰 spec 的产品经理也能做到。

**架构师**：短期安全。AI 做规划还需要人审批。但 ULTRAPLAN（30 分钟远程 Opus 规划）和 PlanAgent（自称"软件架构师"）在逐步侵蚀这个领域。

### 真正的分化轴

**不是"会不会写代码"，是"有没有系统性判断力"。**

具体说：
- 能不能判断 AI 的方案方向是否正确？
- 30 个 PR 里失败的 2 个是不是关键路径？
- AI 说 PARTIAL（"我验证不了"）是什么意思？
- 92% 成功率背后那 8% 致不致命？

一个有系统思维的产品经理，可能比一个只会写代码的初级程序员更能有效使用这套工具。**"会写代码 = 技术人才"这个等式正在失效。**

### AI 输出的欺骗性

对不懂技术的人，AI 的输出有三层误导：

1. **最危险**：`/batch` 报告"24 个任务完成了 22 个"。听起来 92% 成功率很好，但失败的 2 个可能是最关键的模块。外行看不出来。
2. **中等**：验证 agent 的 PARTIAL 判定。技术上意思是"环境受限我验证不了"，但外行很可能理解为"大体通过，有小问题"。
3. **较隐蔽**：coordinator 的 synthesis 环节不透明。用户看到结论，但看不到 AI 是怎么理解研究结果、做了哪些假设的。技术专家会质疑假设，外行会直接信任结论。

这会造成 **"掌握 AI 的技术精英"和"被 AI 输出误导的外行"之间的鸿沟**。Claude Code 有一些防护机制（verification agent 被训练识别自己的借口、plan 需要审批、危险模式有红色警告），但这些都是 opt-in 的——只有懂得启用它们的人才能获益。

---

## 修正记录

Doodlestein 挑战修正了以下表述：
- VERIFICATION_AGENT：~~生产就绪~~ → 代码完整，内部 A/B 测试中
- ULTRAPLAN：~~本地-云端混合~~ → 远端执行，本地只负责发送和接收
- autoDream：~~在你睡觉时整理记忆~~ → 代码完整但默认关闭
- BYOC：~~范式转变~~ → 战略意图信号
- Multi-Agent Protocol：~~可提取~~ → 不建议，MCP 会自己做

## 下一步

- 如果要做 AI 记忆统一层 → `/ae:plan`
- 想深挖某个子系统（KAIROS 架构、autoDream 优化）→ `/ae:think`
- 跟踪 MCP agent coordination 标准进展 → 每周 15 分钟，零代码投入
