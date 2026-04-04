# Claude Code 长时间后台任务指南

如何在不影响主 session 的情况下运行长时间后台任务：定时监听、事件响应、自动处理。

---

## 可用机制总览

| 机制 | 阻塞主 session | 跨 session | 需要进程存活 | 激活方式 |
|------|---------------|-----------|-------------|---------|
| Cron（session 内） | 否（turn 间隙） | 否 | 是 | `CronCreate` / `/loop` |
| Cron（durable） | 否（turn 间隙） | 数据持久化 | 是（执行需要） | `CronCreate` + `durable: true` |
| Agent `run_in_background` | 否（独立异步） | 否 | 是 | Agent 工具参数 |
| Agent `isolation: worktree` | 否 | worktree 残留 | 是 | Agent 工具参数 |
| Auto-background（120s） | 否（自动转后台） | 否 | 是 | `CLAUDE_AUTO_BACKGROUND_TASKS=1` |
| Print mode + resume | N/A（headless） | 是 | 否（按需启动） | `claude -p --resume` |
| 外部 crontab + print mode | N/A | 是 | 否（系统调度） | `crontab -e` |
| GitHub Actions | N/A | 是 | 否（云端运行） | `.github/workflows/` |
| tmux 保活 | 否 | 是 | tmux server 存活 | `tmux new-session -d` |

**核心理解**：Claude Code 的 cron/background agent 全部是**进程内机制**。没有 Claude Code 进程在运行时，durable cron 只是磁盘上的 JSON 文件，不会自动执行。要实现真正的无人值守，需要外部调度器（crontab/systemd/tmux）来保证有进程在跑。

---

## 方案一：Session 内 Cron + Background Agent + Worktree

**最简单的方案**——全部在 session 内配置，适合"开着 Claude Code 工作时顺便监听"。

### 架构图

```
┌─ Claude Code 主 Session（你正常工作）────────────────────────┐
│                                                              │
│  ┌─ Cron Scheduler（1 秒 tick，.unref()）──────────────────┐ │
│  │                                                          │ │
│  │  每 N 分钟检查 scheduled_tasks                           │ ��
│  │    ↓                                                     │ │
│  │  到期 → 将 prompt 注入 messageQueue (priority: 'later')  │ │
│  │                                                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                        ↓                                      │
│  ┌─ REPL Turn 间隙 ──────────────────────────────────────────┐│
│  │                                                            ││
│  │  你的 turn 完成 → 检查 queue → 取出 cron prompt → 执行    ││
│  │    ↓                                                       ││
│  │  模型: Bash(gh pr list ...) → 发现需要处理                 ││
│  │    ↓                                                       ││
│  │  模型: Agent(run_in_background: true,                      ││
│  │              isolation: "worktree",                         ││
│  │              prompt: "处理 PR #42 review comments")        ││
│  │                                                            ││
│  └────────────────────────────────────────────────────────────┘│
│                        ↓                                      │
│  ┌─ Background Agent（独立异步循环）──────────────────────────┐│
│  │                                                            ││
│  │  CWD: .claude/worktrees/agent-abc12345/                    ││
│  │  Branch: agent-branch-abc12345                             ││
│  │                                                            ││
│  │  1. Bash: gh pr view 42 --comments --json body             ││
│  │  2. Read/Edit: 修改代码                                    ││
│  │  3. Bash: git add . && git commit -m "fix review"          ││
│  │  4. Bash: git push origin agent-branch                     ││
��  │  5. 完成 → enqueueAgentNotification()                      ││
│  │           → <task-notification> 注入主 session queue        ││
│  │                                                            ││
│  └────────────────────────────────────────────────────────────┘│
│                                                              │
│  你继续正常工作，不受干扰                                      │
│  下一个 turn 开始时看到 agent 完成通知                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 设置步骤

**1. 创建监听任务**

自然语言方式：

```
帮我设置一个定时任务：每 5 分钟检查一下这个 repo 的 open PRs，
如果有新的 review comment 需要处理，起一个 background agent 
在 worktree 里修改代码并 push。
```

或者用 `/loop` skill：

```
/loop 5m 用 gh pr list 检查 open PRs 的 review 状态，如果有 changes_requested，起一个 background worktree agent 来处理 review comments 并 push fix
```

**2. 模型执行的实际流程**

模型收到 cron 注入的 prompt 后：

```
步骤 1 — 检查状态
  Bash: gh pr list --state open --json number,reviewDecision,url

步骤 2 — 如果无需处理
  返回空结果，turn 结束（低开销，几百 token）

步骤 3 — 如果有需要处理的 PR
  Agent(
    run_in_background: true,     ← 不阻塞主 session
    isolation: "worktree",        ← 独立 git 工作目录
    prompt: "PR #42 有 review comments 需要处理。
             1. gh pr view 42 --comments --json body,author
             2. 分析每条 comment，修改对应代码
             3. git add . && git commit -m 'address review comments'
             4. git push origin HEAD
             5. gh pr comment 42 --body '已处理 review comments'"
  )

步骤 4 — 主 session 立即返回
  模型: "已启动 background agent 处理 PR #42，你可以继续工作。"
```

**3. Durable 模式（跨 session 持久化）**

```
创建一个 durable 定时任务（写入 .claude/scheduled_tasks.json），
每 5 分钟检查 PR 状态并处理
```

Durable cron 的 JSON 格式（`.claude/scheduled_tasks.json`）：

```json
{
  "tasks": [
    {
      "id": "uuid-xxx",
      "cron": "*/5 * * * *",
      "prompt": "检查 open PRs 的 review 状态...",
      "createdAt": 1775320000000,
      "lastFiredAt": 1775325000000,
      "recurring": true
    }
  ]
}
```

### 关键细节

**Cron 执行时机**：

- Scheduler 每 1 秒 tick 一次（`setInterval`，`.unref()` 不阻止进程退出）
- 到期的 cron 注入为 user turn，`priority: 'later'`
- 如果你正在对话（turn in progress），cron 排队等待
- 只在 turn 间隙执行——**不会打断你正在进行的对话**
- Jitter：recurring 任务最多延迟 10% 周期（上限 15 分钟），防止整点拥堵

**Background Agent 生命周期**：

```
创建 → registerAsyncAgent() → isBackgrounded: true
  ↓
独立异步循环运行（同进程，不同 async context）
  ↓
主 session 的 tool call 返回 { status: 'async_launched' }
  ↓
Agent 工作中... 主 session 继续交互
  ↓
Agent 完成 → enqueueAgentNotification()
  ↓
<task-notification> 注入主 session 消息队列
  包含：status, output_file_path, usage_stats
  ↓
模型在下一个 turn 看到通知，可以 Read output_file 查看完整结果
```

**Auto-background（120s 自动转后台）**：

如果一个 foreground agent 运行超过 120 秒，自动转为 background。需要启用：

```bash
export CLAUDE_AUTO_BACKGROUND_TASKS=1
```

也可以在 session 中手动按 `Ctrl+B` 将当前 foreground agent 转后台。

**Worktree 生命周期**：

```
创建:
  git worktree add .claude/worktrees/agent-<id[:8]>/ -b agent-branch-<id[:8]>

Agent 工作:
  CWD 覆盖为 worktree 路径
  所有 Bash/Read/Edit/Write 操作在 worktree 内执行
  主工作目录完全不受影响

完成后:
  hasWorktreeChanges() 检查：
    - 无变更（no uncommitted changes, no new commits）→ 自动删除 worktree + 分支
    - 有变更 → 保留 worktree，路径 + 分支名报告给主 session

独立 commit/push:
  Agent 有完整 Bash 权限，可以在 worktree 内执行：
    git add . && git commit -m "fix" && git push origin HEAD
  这些操作与主分支完全隔离
```

**多 session Cron 锁**：

```
Session A (拥有锁):
  cronTasksLock.ts → 创建锁文件
  → 正常执行 durable cron

Session B (同一目录):
  检测到锁文件已被 Session A 持有
  → 每 5 秒探测锁状态
  → 如果 Session A 崩溃（锁文件 stale）→ Session B 接管

Session A 正常退出:
  释放锁 → Session B 接管
```

---

## 方案二：外部脚本 + Print Mode + Session Resume

**真正的无人值守**——不依赖 Claude Code 进程存活，由系统调度器保证执行。

### 架构图

```
┌─ 系统调度器（crontab / launchd / systemd）──┐
│                                              │
│  */5 * * * * bash ~/.scripts/pr-monitor.sh   │
│                                              │
└──────────────┬───────────────────────────────┘
               ↓
┌─ pr-monitor.sh ──────────────────────────────┐
│                                              │
│  1. gh pr list → 检查是否有需要处理的 PR      │
│  2. 无 → exit 0（零开销）                     │
│  3. 有 → 启动 claude -p（headless 单轮执行）  │
│     - --resume <saved-session-id>             │
│     - --output-format stream-json --verbose   │
│  4. 保存 session ID 供下次复用                 │
│  5. claude -p 执行完 → 进程退出               │
│                                              │
└──────────────────────────────────────────────┘
```

### 完整实现

**监听脚本**（`~/.scripts/pr-monitor.sh`）���

```bash
#!/bin/bash
set -euo pipefail

REPO_DIR="/path/to/your/repo"
SESSION_FILE="$HOME/.claude/pr-monitor-session"
LOG_FILE="$HOME/.logs/pr-monitor.log"
LOCK_FILE="/tmp/pr-monitor.lock"

# 防止并发执行
exec 200>"$LOCK_FILE"
flock -n 200 || { echo "$(date): already running, skipping" >> "$LOG_FILE"; exit 0; }

cd "$REPO_DIR"

# 拉取最新代码
git fetch origin --quiet

# 检查是否有需要处理的 PR
PRS=$(gh pr list --state open --json number,reviewDecision,headRefName \
  --jq '.[] | select(.reviewDecision == "CHANGES_REQUESTED")')

if [ -z "$PRS" ]; then
  echo "$(date): no PRs need attention" >> "$LOG_FILE"
  exit 0
fi

echo "$(date): found PRs needing attention" >> "$LOG_FILE"

# 遍历需要处理的 PR
echo "$PRS" | jq -r '.number' | while read PR_NUMBER; do
  BRANCH=$(echo "$PRS" | jq -r "select(.number == $PR_NUMBER) | .headRefName")
  
  echo "$(date): processing PR #$PR_NUMBER (branch: $BRANCH)" >> "$LOG_FILE"
  
  # 构建 resume 参数
  RESUME_ARGS=""
  if [ -f "$SESSION_FILE" ]; then
    SESSION_ID=$(cat "$SESSION_FILE")
    RESUME_ARGS="--resume $SESSION_ID"
  fi
  
  # 执行 Claude Code（headless）
  OUTPUT=$(claude -p $RESUME_ARGS \
    --output-format stream-json --verbose \
    "PR #$PR_NUMBER (branch: $BRANCH) 收到了 changes_requested。
     1. git checkout $BRANCH && git pull origin $BRANCH
     2. gh pr view $PR_NUMBER --comments --json body,author,path,line
     3. 逐条分析 review comments，修改对应代码
     4. git add . && git commit -m 'address review feedback for PR #$PR_NUMBER'
     5. git push origin $BRANCH
     6. gh pr comment $PR_NUMBER --body '✅ Review feedback addressed. Please re-review.'
     7. git checkout main" 2>> "$LOG_FILE") || true
  
  # 保存 session ID
  NEW_SESSION=$(echo "$OUTPUT" | jq -r 'select(.type=="system") | .session_id' 2>/dev/null | head -1)
  if [ -n "$NEW_SESSION" ]; then
    echo "$NEW_SESSION" > "$SESSION_FILE"
  fi
  
  echo "$(date): finished PR #$PR_NUMBER" >> "$LOG_FILE"
done
```

**安装**：

```bash
# 创建日志目录
mkdir -p ~/.logs

# 设置执行权限
chmod +x ~/.scripts/pr-monitor.sh

# 添加到 crontab
crontab -e
# 添加：
# */5 * * * * bash ~/.scripts/pr-monitor.sh
```

**macOS launchd 版本**（比 crontab 更可靠）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.pr-monitor</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/ckai/.scripts/pr-monitor.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>300</integer>
    <key>StandardOutPath</key>
    <string>/Users/ckai/.logs/pr-monitor-stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/ckai/.logs/pr-monitor-stderr.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>ANTHROPIC_API_KEY</key>
        <string>sk-ant-xxx</string>
    </dict>
</dict>
</plist>
```

```bash
# 安装
cp com.user.pr-monitor.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.user.pr-monitor.plist

# 卸载
launchctl unload ~/Library/LaunchAgents/com.user.pr-monitor.plist
```

### Print Mode 详细说明

**基本用法**：

```bash
# 单轮 headless 执行
claude -p "检查 PR 状态"

# 从 stdin 传入 prompt
echo "检查 PR 状态" | claude -p

# 恢复已有 session（保留上下文）
claude -p --resume <session-uuid> "继续处理"

# 恢复最近的 session
claude -p --continue "继续处理"

# 从指定消息恢复（部分回滚）
claude -p --resume <session-uuid> --resume-session-at <message-uuid> "从这里重来"
```

**Output 格式**：

```bash
# 默认：纯文本到 stdout
claude -p "hello"

# Stream JSON（NDJSON，包含 session_id）
claude -p --output-format stream-json --verbose "hello"
# 输出：
# {"type":"system","session_id":"uuid-xxx",...}
# {"type":"turn","content":"..."}
# {"type":"result","content":"..."}
```

**链式调用（scripting pattern）**：

```bash
# 第一次：获取 session ID
SESSION=$(claude -p --output-format stream-json --verbose "分析 auth 模块" \
  | jq -r 'select(.type=="system") | .session_id' | head -1)

# 后续：在同一 session 中继续
claude -p --resume "$SESSION" "根据分析结果，修改 auth.ts"
claude -p --resume "$SESSION" "写测试"
claude -p --resume "$SESSION" "运行测试并修复失败"
```

**Cron 在 print mode 中的行为**：

Scheduler 的 `setInterval` 设了 `.unref()`，所以 **不会阻止 `-p` 模式的进程退出**。Cron 只在 turn 进行中时触发（如果恰好 tick 到了），`-p` 完成后进程正常退出。

---

## 方案三：tmux 保活 + Session 内 Cron

**最省心的无人值守**——利用 tmux 保持 Claude Code 进程存活。

### 架构图

```
┌─ tmux server（后台常驻）──────────────────────┐
│                                                │
│  Session "claude-monitor"                      │
│    └─ Claude Code 进程（持续运行）              │
│         ├─ Durable cron scheduler（活跃）       │
│         ├─ Background agents（按需启动）        │
│         └─ REPL（可 attach 交互）               │
│                                                │
└────────────────────────────────────────────────┘

你的终端：
  tmux attach -t claude-monitor  → 进入交互
  Ctrl+B d                       → detach，进程继续跑
  关闭终端                        → tmux server 不受影响
```

### 实现

```bash
# 启动（在项目目录下）
cd /path/to/your/repo
tmux new-session -d -s claude-monitor "claude --name monitor"

# 进入交互（设置 cron、检查状态）
tmux attach -t claude-monitor

# 在 Claude Code 中设置 durable cron
> 创建 durable recurring cron，每 5 分钟检查 PR 状态并处理

# detach（Ctrl+B d 或 Ctrl+A d 如果你的 prefix 是 C-a）
# 进程继续跑，cron 继续触发

# 查看状态（不 attach）
tmux capture-pane -t claude-monitor -p | tail -20

# 停止
tmux kill-session -t claude-monitor
```

### 优势

- Durable cron + 活进程 = 真正的定时执行
- 随时 attach 交互，检查状态，手动触发任务
- tmux-resurrect 插件可以在系统重启后恢复 tmux session（但 Claude Code 进程需要重新启动）

---

## 方案四：GitHub Actions + Claude Code

**完全事件驱动**——webhook 触发，云端执行，零本地资源。

### PR Review 自动处理

```yaml
# .github/workflows/pr-review-handler.yml
name: Auto-handle PR review feedback
on:
  pull_request_review:
    types: [submitted]

jobs:
  handle-changes-requested:
    if: github.event.review.state == 'changes_requested'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: write
      pull-requests: write
    
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure git
        run: |
          git config user.name "Claude Code Bot"
          git config user.email "bot@example.com"

      - name: Handle review comments
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          npx @anthropic-ai/claude-code -p \
            "PR #${{ github.event.pull_request.number }} 收到 changes_requested review。
             1. 用 gh pr view ${{ github.event.pull_request.number }} --comments 读取所有 review comments
             2. 逐条分析，修改对应代码
             3. git add . && git commit -m 'address review feedback'
             4. git push origin HEAD
             5. gh pr comment ${{ github.event.pull_request.number }} --body '✅ Review feedback addressed automatically.'"
```

### CI 失败自动修复

```yaml
# .github/workflows/auto-fix-ci.yml
name: Auto-fix CI failures
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]

jobs:
  auto-fix:
    if: github.event.workflow_run.conclusion == 'failure'
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions:
      contents: write
      actions: read
    
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_branch }}
          fetch-depth: 0

      - name: Get failure logs
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh run view ${{ github.event.workflow_run.id }} --log-failed > /tmp/ci-failure.log

      - name: Attempt auto-fix
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          npx @anthropic-ai/claude-code -p \
            "CI 失败了。失败日志在 /tmp/ci-failure.log。
             1. 读取日志，分析失败原因
             2. 修改代码修复问题
             3. git add . && git commit -m 'fix: auto-fix CI failure'
             4. git push origin HEAD"
```

---

## 实用模式

### 模式 1：PR 状态监听 + 自动处理

```
/loop 5m 检查所有 open PR 的 review 状态：
1. gh pr list --state open --json number,reviewDecision,url
2. 对每个 reviewDecision=="CHANGES_REQUESTED" 的 PR：
   - 起一个 background worktree agent
   - agent 读取 gh pr view --comments
   - 修改代码回应 review
   - git commit + git push
3. 对每个 review 通过的 PR，报告一下状态
```

### 模式 2：CI 状态监听 + 自动修复

```
/loop 10m 检查最近的 CI 运行：
1. gh run list --limit 5 --json status,conclusion,headBranch
2. 如果有 failure：
   - gh run view <id> --log-failed
   - 分析失败原因
   - 起 background worktree agent 修复并 push
```

### 模式 3：Issue 监听 + 自动分析

```
/loop 15m 检查新 issue：
1. gh issue list --state open --json number,title,createdAt --jq '[.[] | select(.createdAt > "2026-04-04")]'
2. 对每个新 issue：
   - 读取 issue 内容
   - 分析涉及的代码
   - 在 issue 下评论初步分析结果
```

### 模式 4：多 PR 并行处理

```
帮我同时处理这 3 个 PR 的 review comments：
- PR #41：起一个 background worktree agent
- PR #42：起一个 background worktree agent  
- PR #43：起一个 background worktree agent
每个 agent 独立在自己的 worktree 里修改、commit、push。完成后通知我。
```

### 模式 5：依赖更新 + 自动测试

```
/loop 1d 每天检查一次依赖更新：
1. npm outdated --json 或 cargo outdated
2. 如果有需要更新的：
   - 起 background worktree agent
   - 更新依赖
   - 运行测试
   - 测试通过 → commit + push + 创建 PR
   - 测试失败 → 回滚，通知主 session
```

---

## Session 断了怎么恢复

### Durable Cron 的生命周期（关键理解）

```
Session 运行中:
  cronScheduler.ts (setInterval, 1s tick)
    ↓ 读取
  .claude/scheduled_tasks.json
    ↓ 到期
  注入 user turn 执行

Session 退出:
  setInterval 停止
  .claude/scheduled_tasks.json 保留在磁盘
  → 没有进程读取它，没有任何东西执行
  → cron 只是数据，不是守护进程

新 Session 启动:
  cronScheduler.ts 重新启动
    ↓ 读取
  .claude/scheduled_tasks.json
    ↓ 检测 missed tasks
  一次性任务 → AskUserQuestion("这些任务错过了，要执行吗？")
  Recurring 任务 → 不提示，下一个 tick 正常触发
```

### 各层恢复机制

| 组件 | Session 断了会怎样 | 恢复方式 |
|------|------------------|---------|
| 主对话 | 自动保存在 `~/.claude/sessions/*.jsonl` | `claude --continue` |
| 中断的 turn | 检测到 `interrupted_turn` | `--resume` 时自动注入 "Continue from where you left off." |
| Session 内 cron | **丢失**（仅在内存） | 需要重新创建 |
| Durable cron | **存活**（`.claude/scheduled_tasks.json`） | 下次启动自动恢复 |
| Background agent | **死亡**（同进程） | 可通过 `resumeAgentBackground()` 恢复 |
| Worktree | **保留在磁盘** | agent 恢复时自动检测并复用 |

### 恢复流程

**1. 恢复主 session**

```bash
# 恢复最近的 session
claude --continue

# 或指定 session ID
claude --resume <session-uuid>

# 分叉恢复（不污染原 session）
claude --resume <session-uuid> --fork-session --name "recovery"
```

Session 恢复时自动检测 `interrupted_turn`，注入 "Continue from where you left off." 让模型从断点继续。

**2. Durable Cron 自动恢复**

启动时 scheduler 自动读取 `.claude/scheduled_tasks.json`：

- **Recurring 任务**：无提示，下一个 tick 正常触发（`lastFiredAt` 用于计算下次触发时间，避免重复）
- **一次性 missed 任务**：通过 `AskUserQuestion` 询问确认后执行，然后从文件删除
- **多 session 锁**：同一目录下只有一个 session 拥有 scheduler，其余每 5 秒探测接管

**3. Background Agent 恢复**

`resumeAgentBackground()`（`src/tools/AgentTool/resumeAgent.ts`）：

```
1. 读取 agent transcript（~/.claude/sessions/ 中的 agent 子 session）
2. 读取 agent metadata（worktree 路径、agent 类型、描述）
3. 重建消息历史：
   - 过滤孤立的 thinking-only 消息
   - 过滤未完成的 tool use（中断导致的残留）
   - 重建 content replacement state（大工具结果的磁盘引用）
4. 检查 worktree：
   - 存在 → 恢复到同一 worktree + 更新 mtime（防止被 stale cleanup 删掉）
   - 不存在 → fallback 到主 CWD
5. 重新注册为 async agent → 后台继续执行
```

实际操作：

```bash
claude --continue
# 然后：
> 上次有 background agent 在处理 PR #42，session 断了，帮我恢复
```

**4. 手动检查残留 worktree**

```bash
# 列出所有残留 worktree
ls -la .claude/worktrees/

# 检查某个 worktree 的状态
cd .claude/worktrees/agent-xxx
git status
git log --oneline -5
git diff

# 手动清理（如果不再需要）
cd /path/to/repo
git worktree remove .claude/worktrees/agent-xxx
```

### 最佳实践：自愈流程

创建一个 durable cron，prompt 包含恢复逻辑：

```
创建 durable recurring cron（每 5 分钟），prompt 如下：

1. 检查 .claude/worktrees/ 是否有未完成的 agent worktree
   - 有 → 检查 worktree 状态，如果有未 push 的 commit，恢复该 agent 继续工作
2. gh pr list 检查 open PRs 的 review 状态
   - 有 changes_requested → 起新的 background worktree agent 处理
3. 如果没有需要处理的，静默结束（不要输出任何内容）
```

这样即使 session 断了：

```
claude --continue
  ↓
Durable cron 自动恢复
  ↓ 第一次 tick
检测到残留 worktree → 恢复未完成的 agent
  ↓
继续定时检查 PR 状态
  ↓
整套流程自愈
```

---

## 方案对比

| | 方案一 Session 内 Cron | 方案二 外部脚本 | 方案三 tmux 保活 | 方案四 GitHub Actions |
|---|---|---|---|---|
| 复杂度 | 低 | 中 | 低 | 中 |
| 需要进程存活 | 是 | 否 | 是（tmux） | 否 |
| 事件驱动 | 轮询 | 轮询或 webhook | 轮询 | webhook |
| Context 连续性 | 共享主 session | `--resume` 可选 | 共享 session | 无（每次独立） |
| 不阻塞正常工作 | cron 在 turn 间隙 | 完全独立进程 | 独立 tmux session | 云端 |
| 跨 session | durable cron | 天然 | 天然 | 天然 |
| Token 消耗 | 共享 context（低） | 每次独立（高） | 共享 context（低） | 每次独立（高） |
| 适合场景 | 开发时顺便监听 | 无人值守自动化 | 长时间自治运行 | CI/CD 集成 |

---

## 注意事项

### Cron 的限制

- **不是独立进程**：Cron 任务注入为主 session 的 user turn，在 turn 间隙执行。连续不停对话时 cron 排队。
- **共享 context**：Cron turn 和正常对话共享 context window。频繁触发加速 autocompact。
- **最小间隔 1 分钟**：Cron 表达式最小粒度是分钟。
- **Session 内 cron 不持久**：默认创建的 cron 退出即消失。用 `durable: true` 持久化。
- **Recurring 默认 7 天过期**：除非标记 `permanent: true`（只能直接编辑 JSON 文件）。
- **Durable cron 不是守护进程**：只是磁盘上的 JSON 文件。没有 Claude Code 进程运行时不会执行。

### Background Agent 的限制

- **同进程**：在主 session 同一个 Node.js 进程中运行。进程退出则所有 background agent 终止。
- **资源竞争**：多个 background agent 共享 CPU 和内存。大量并行可能影响主 session 响应。
- **Token 消耗**：每个 background agent 独立调用 API。

### Worktree 的限制

- **磁盘空间**：每个 worktree 是完整 git checkout。大型 repo 中并行多个会占大量磁盘。
- **分支冲突**：多个 worktree agent 修改同一区域，push 时可能冲突。
- **清理**：有变更的 worktree 不会自动删除。需手动 `git worktree remove`。

### Print Mode 的限制

- **单轮执行**：`-p` 执行一轮后退出。不能中途交互。
- **无 context（不 resume 时）**：全新 session 没有历史上下文。
- **认证**：需要 `ANTHROPIC_API_KEY` 环境变量或 OAuth token。

---

## 环境变量参考

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_DISABLE_CRON=1` | 禁用所有 cron | 未设置（cron 开启��� |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` | 禁用 `run_in_background` 参数 | 未设置 |
| `CLAUDE_AUTO_BACKGROUND_TASKS=1` | 启用 120s 自动转后台 | 未设置 |
| `ANTHROPIC_API_KEY=sk-ant-xxx` | API 认证（print mode / 外部脚本需要） | 未设置 |

## 文件位置参考

| 文件 | 用途 |
|------|------|
| `.claude/scheduled_tasks.json` | Durable cron 任务存储 |
| `.claude/worktrees/` | Agent worktree 工作目录 |
| `~/.claude/sessions/` | Session transcript 存储（包含 agent 子 session） |
| `~/.claude/tasks/` | Team task list 存储 |
