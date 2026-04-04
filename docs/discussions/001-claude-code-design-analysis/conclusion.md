---
id: "001"
title: "Claude Code Design Analysis — Conclusion"
concluded: 2026-04-04
plan: ""
---

# Claude Code Design Analysis — Conclusion

## Decision Summary

| # | Topic | Key Finding |
|---|-------|-------------|
| 1 | 设计亮点 | 6 个值得借鉴的模式（buildTool factory、并发分区、三层权限、deferred tool loading、prompt cache 稳定性、autocompact 断路器） |
| 2 | 战略布局 | KAIROS 是核心赌注（编码助手→持久自主代理），MCP registry 是平台控制面，IDE bridge 是状态连续性护城河 |
| 3 | 省钱技巧 | 关 fast mode（6x）、维护稳定 CLAUDE.md 利用 cache（10x）、搜索型子任务用 haiku |

## 设计亮点（可借鉴）

1. **`buildTool` 工厂** — plain object + factory + typed defaults，fail-closed 安全默认值
2. **工具并发分区** — read 并行 / write 串行，context modifier 队列化后处理
3. **三层权限系统** — deny 在注册时生效（模型看不到被禁工具），细粒度 pattern match + ML 分类器
4. **Deferred Tool Loading** — MCP 工具按需拉取 schema，不预占 context
5. **Prompt Cache 稳定性工程** — 工具排序保证 cache key 稳定，15 维 cache break 检测
6. **Autocompact 断路器** — 3 次连续失败后跳过，数据驱动（250K API calls/day 浪费）

**挑战者反驳**：Tool 接口 ~35 方法违反 ISP；启动优化是对臃肿启动图的补偿；QueryEngine 和 REPL 跑在不同代码路径（技术债）。

## 战略布局

| 阶段 | Feature Flag | 含义 |
|------|-------------|------|
| 编码助手 | 当前 shipping | 交互式 CLI |
| 多代理协作 | COORDINATOR_MODE + FORK_SUBAGENT + TEAMMEM | 多 agent 协同 |
| 持久自主代理 | KAIROS + PROACTIVE + DAEMON | 后台常驻，主动执行 |
| 远程运行时 | BRIDGE_MODE + CCR_AUTO_CONNECT | 远程机器 session |
| 环境计算 | KAIROS_CHANNELS + KAIROS_GITHUB_WEBHOOKS | 全面感知开发环境 |

MCP registry（`api.anthropic.com/mcp-registry/v0/servers`）是中心化信任控制。XAA（Cross-App Access）是企业 IdP 锁定。

## 省钱要点

| 操作 | 效果 |
|------|------|
| `CLAUDE_CODE_DISABLE_FAST_MODE=1` | 6x 省钱 |
| 保持 CLAUDE.md 稳定 | Cache read 便宜 10x |
| 子任务用 `model: "haiku"` | $1/$5 vs Opus $15/$75 |
| 不频繁增减 MCP 工具 | 避免 cache bust（77% 的 break 原因） |
| `CLAUDE_CODE_SIMPLE=1` | 极简模式，砍 system prompt |

## 后续研究（本 session 已完成）

此 analysis 引出了以下深入研究，均已完成并有独立文档：
- [002] Agent Teams 替代方案
- [003] 跨 session 通讯方案（文件 IPC + Redis MCP + 动态调度）
- [004] OpenClaw 精简版架构（Go + UDS + JSON-RPC）
- [005] Claw <-> Claude Code 桥接

## Team Composition (Analysis Phase)

| Agent | Role | Backend |
|-------|------|---------|
| TL | Moderator + Synthesizer | Claude |
| archaeologist | Deep code dive | Claude |
| standards-expert | Industry comparison | Claude |
| challenger | Blind spot review + opposition | Claude |
| codex-proxy | OpenAI-family perspective | Codex (o4-mini) |
| gemini-proxy | Google-family perspective | Gemini (quota-limited) |
