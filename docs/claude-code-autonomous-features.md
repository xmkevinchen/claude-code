# Claude Code 持久自主代理功能深挖

不修改源代码，外部用户能用什么、不能用什么、以及可替代方案。

---

## 核心结论

`feature()` 是 Bun 构建时的死代码消除——不是运行时开关。外部构建中被禁用的 feature，其代码已被物理移除，**任何 env var、CLI flag、config 都无法激活**。

但有些功能是无条件发布的，或通过 GrowthBook 服务端 gate 控制（默认开启），这些是外部用户可以直接使用的。

---

## 功能可用性总览

| 功能 | 外部可用? | 激活方式 |
|------|----------|---------|
| Session resume / continue | **YES** | `--continue` / `--resume <id>` |
| Cron 定时任务 (/loop) | **YES** | CronCreate 工具，默认开启 |
| `claude server` (HTTP/WS) | **YES** | `claude server --unix <path>` |
| Multi-agent (AgentTool) | **YES** | Agent 工具，无条件发布 |
| Remote Control (桥接) | **订阅者** | `/remote-setup`，需 claude.ai 订阅 |
| Coordinator 模式 | **可能** | `CLAUDE_CODE_COORDINATOR_MODE=1`（需测试） |
| KAIROS 守护进程 | NO | 构建时移除 |
| --proactive / SleepTool | NO | 构建时移除 |
| Background sessions (--bg) | NO | 构建时移除 |
| UDS 点对点通信 | NO | 构建时移除，ant-only |
| CCR 自动连接 | NO | 构建时移除，ant-only |

---

## 可用功能详解

### 1. Session 链式执行（最接近"持久代理"的外部方案）

**完全可用，无任何 gate。**

```bash
# 继续当前目录最近的 session
claude --continue

# 指定 session ID 恢复
claude --resume <session-uuid>

# 从某个 session 分叉出新 session（保留上下文，新 ID）
claude --resume <session-uuid> --fork-session

# 给 session 命名（方便后续找到）
claude --name "feature-x-implementation"

# 创建时指定 UUID（方便脚本化工作流）
claude --session-id <your-uuid>

# 恢复到特定消息（部分回滚）
claude --resume-session-at <message-id>

# 恢复与某个 PR 关联的 session
claude --from-pr 123
```

**实践模式：长任务分段执行**

```bash
# 第一天：开始任务
claude --name "refactor-auth" -p "分析 auth 模块，列出需要重构的点"

# 第二天：继续
claude --continue  # 自动恢复 refactor-auth session，保留完整上下文

# 或者分叉：保留原始分析，开始新的实现分支
claude --resume <id> --fork-session --name "refactor-auth-v2"
```

Session 数据保存在 `~/.claude/sessions/`，`.jsonl` 格式，可跨目录恢复。中断的 turn 会通过 `conversationRecovery` 自动恢复。

---

### 2. Cron 定时任务（/loop skill）

**默认开启**，GrowthBook gate `tengu_kairos_cron` 默认 `true`，仅作为全局紧急开关。

```bash
# 每 5 分钟运行一次检查
/loop 5m "检查 CI 状态，如果失败了分析原因"

# 默认 10 分钟间隔
/loop "检查 deploy 日志有没有异常"
```

底层工具：
- `CronCreate` — 创建定时任务
- `CronList` — 列出所有定时任务
- `CronDelete` — 删除定时任务

持久化位置：`~/.claude/scheduled_tasks.json`

禁用方式：
```bash
export CLAUDE_CODE_DISABLE_CRON=1
```

**注意**：`RemoteTriggerTool`（`/schedule` skill，服务端远程定时代理）需要 claude.ai 订阅 + GrowthBook 灰度开放，不是所有用户都能用。

---

### 3. `claude server` — HTTP/WebSocket Session 服务

**无条件发布**，最接近"后台服务"的外部方案。

```bash
# 启动 HTTP session server
claude server

# 通过 Unix socket 启动（适合本地 IPC）
claude server --unix /tmp/claude.sock
```

这提供了一个 HTTP/WebSocket API 来创建和管理 Claude Code session。可以：
- 从脚本或其他程序调用
- 保持 session 持久运行
- 多客户端连接同一个 session

**实践模式：简易后台代理**

```bash
# Terminal 1: 启动 server
claude server --unix /tmp/claude-agent.sock &

# Terminal 2: 通过 API 发送任务
# （具体 API 格式需参考 bridge 协议）
```

---

### 4. Multi-agent 协作（AgentTool）

**无条件发布**。AgentTool、SendMessageTool、TaskCreate/Update/List 都是标准工具。

```bash
# 在 prompt 中请求多 agent 协作
claude -p "用 3 个 agent 并行：一个搜索 bug，一个写测试，一个写修复"
```

支持的 agent 隔离模式：
- **默认（in-process）**：共享内存，轻量
- **`isolation: "worktree"`**：独立 git worktree，文件隔离，防止冲突

---

### 5. Coordinator 模式（需测试）

如果 `feature('COORDINATOR_MODE')` 在外部构建中为 `true`：

```bash
export CLAUDE_CODE_COORDINATOR_MODE=1
claude -p "实现 feature X"
```

Coordinator 模式会注入编排者系统 prompt，让 Claude 自动分解任务、分配给 worker agent、管理团队生命周期。比普通 AgentTool 多了：
- 结构化的 coordinator system prompt
- `TeamCreate`/`TeamDelete` 工具
- Worker 工具白名单（防止递归 agent 树）
- 可选的 scratchpad 目录

**验证方法**：设置环境变量后启动 session，看 Claude 是否自动以 coordinator 身份行动。如果不生效，说明该 feature 在外部构建中被移除了。

---

### 6. Remote Control（claude.ai 订阅者）

```bash
# 设置远程控制
/remote-setup

# 或在 settings 中启用启动时自动连接
# remoteControlAtStartup: true
```

允许从桌面端/Web 端连接本地 Claude Code session。需要 claude.ai 订阅 + GrowthBook 灰度开放。

---

## 不可用功能及替代方案

### KAIROS（守护进程 + 主动执行）→ 替代方案

KAIROS 的核心能力是：后台常驻 → 监控事件 → 主动行动 → 推送通知。

**用 cron + session resume 模拟**：

```bash
# crontab -e
# 每 30 分钟运行一次 Claude Code 检查
*/30 * * * * cd /path/to/project && claude --continue --name "monitor" -p "检查 CI 状态和最新 PR，有问题就创建 issue"
```

**用 `claude server` + 外部脚本模拟**：

```bash
#!/bin/bash
# watch-and-act.sh — 简易主动代理
while true; do
    # 检测事件（例如新 PR）
    NEW_PRS=$(gh pr list --state open --json number --jq 'length')
    if [ "$NEW_PRS" -gt "$LAST_COUNT" ]; then
        claude --continue --name "pr-reviewer" -p "review 最新的 PR"
    fi
    LAST_COUNT=$NEW_PRS
    sleep 300  # 5 分钟
done
```

**用 GitHub Actions + Claude Code 模拟**：

```yaml
# .github/workflows/claude-monitor.yml
on:
  pull_request:
    types: [opened, synchronize]
  schedule:
    - cron: '0 */2 * * *'  # 每 2 小时

jobs:
  claude-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx @anthropic-ai/claude-code -p "review this PR" --output-format json
```

### Background Sessions (--bg) → 替代方案

```bash
# 用 tmux/screen 替代
tmux new-session -d -s claude-bg "claude --name 'bg-task' -p '执行长任务...'"

# 后续查看
tmux attach -t claude-bg
```

### Push Notifications → 替代方案

```bash
# 任务完成后发送通知（macOS）
claude -p "完成 feature X" && osascript -e 'display notification "Claude 任务完成" with title "Claude Code"'

# 或用 ntfy.sh
claude -p "完成 feature X" && curl -d "Claude task done" ntfy.sh/my-claude-alerts
```

---

## 架构启示

从 feature flag 分布可以看出 Anthropic 的发布策略：

```
外部用户能用的 ←——————————————————————→ 仅内部
                                            
session resume   cron/loop   AgentTool   coordinator?   remote control   KAIROS   daemon
(无 gate)        (GA)        (无 gate)   (需测试)       (订阅者)         (ant)    (ant)
```

Anthropic 的做法是：
1. **基础设施先行**：session 持久化、agent 通信、cron 调度这些基础能力已经全部就位
2. **渐进式开放**：通过 GrowthBook 灰度控制，功能可以按用户群逐步开放
3. **构建时隔离**：内部实验性功能通过 `feature()` 在构建时完全移除，零泄漏风险

KAIROS 的全面开放只是时间问题——基础设施已经在所有用户的二进制中了（session resume、cron、server），只差上层的主动行为逻辑和守护进程壳。
