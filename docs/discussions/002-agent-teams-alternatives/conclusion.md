---
id: "002"
title: "Agent Teams 替代方案 — Conclusion"
concluded: 2026-04-04
plan: ""
---

# Agent Teams 替代方案 — Conclusion

## Decision Summary

如果 Agent Teams feature 被移除，以下能力仍然可用（源码验证，无 feature gate）：

| 能力 | 来源 | 验证位置 |
|------|------|---------|
| 并行 agent 执行 | `AgentTool` + `run_in_background: true` | AgentTool.tsx:567 |
| 文件隔离 | `isolation: "worktree"` | worktree.ts:902（零 Agent Teams 依赖） |
| 完成通知 | `<task-notification>` XML 回注 | AgentTool.tsx:1171 |
| 定时触发 | `CronCreate` / `/loop` | GA，默认开启 |
| Headless 执行 | `claude -p` + `--resume` | 无 feature gate |
| 并发 session | 多个 `claude -p` 可并行 | 无 session 级别锁 |
| 事件检测 | `FileChanged` hook | fileChangedWatcher.ts:69 |
| 文件读写 | Read/Write/Edit/Bash | 无条件可用 |

**不可用**：SendMessage、TeamCreate/Delete、TaskCreate/Update/List、Mailbox、Idle/Wake。

---

## 通讯能力差距分析

没有 Agent Teams 后，**agent 之间没有真正的通讯能力**。这是最关键的能力缺失。

### Agent Teams 的通讯 vs 没有 Agent Teams

| 通讯模式 | Agent Teams | 无 Agent Teams |
|---------|------------|---------------|
| Agent 间实时通讯 | SendMessage（<1ms 内存 / 500ms mailbox） | **不可能**（没有推送机制） |
| Agent 间异步通讯 | Mailbox 文件 + 500ms 轮询自动唤醒 | 只能靠文件约定，无轮询保证（模型可能忘记检查） |
| Parent -> Child | SendMessage 双向随时 | **不可能**（无法向运行中的 background agent 发消息） |
| Child -> Parent | SendMessage 随时 + idle 通知 | 仅 `<task-notification>`（单向、一次性、完成时） |
| Peer-to-Peer | 直接 SendMessage 互寻址 | **不可能**（background agent 之间无法寻址） |
| 广播 | `to: "*"` 写入所有 mailbox | 不可能 |

### `<task-notification>` 是唯一原生回传

Background agent 完成后自动注入到 parent 的下一个 turn：

```xml
<task-notification>
  status: completed
  output_file: ~/.claude/tasks/xxx/output.jsonl
  usage: { input: 1234, output: 5678 }
</task-notification>
```

**限制**：
- 单向：child -> parent only
- 一次性：只在完成时触发，中途无法通讯
- 无寻址：agent 之间不能互相发 notification

### 不可替代的场景

以下场景**没有 Agent Teams 就做不到**（或质量严重下降）：

| 场景 | 为什么需要 Agent Teams | 无 Agent Teams 的降级方案 |
|------|----------------------|------------------------|
| 多 agent 讨论/辩论（如 ae:discuss） | agent 需要实时互相回应、挑战、迭代立场 | 不可行——串行模拟失去并行独立思考 + 交叉验证的价值 |
| 协作式代码审查（多角度同时审） | reviewer 需要看到彼此的发现避免重复 | 降级为串行审查，每个 reviewer 独立跑 |
| 实时任务重分配 | leader 根据进展动态调整任务分配 | 降级为预分配——所有任务启动前确定，运行中不可调整 |
| Test-fix 循环（一个写测试，一个修代码） | 两个 agent 需要快速来回迭代 | 降级为串行：写测试 -> 跑测试 -> 修代码 -> 再跑测试 |

### 可行的替代通讯模式

对于不需要实时互相通讯的场景，有三个可行模式：

**模式 1：Fan-out + Collect（零基建）**

```
Parent session
  |-- spawn Agent A (background, worktree) --> 独立工作 --> <task-notification>
  |-- spawn Agent B (background, worktree) --> 独立工作 --> <task-notification>
  +-- spawn Agent C (background, worktree) --> 独立工作 --> <task-notification>
                                                              |
                                                    Parent 收到所有通知
                                                    综合 N 个结果
```

Agent 之间完全不通讯。各自独立完成任务，parent 综合。

**模式 2：串行流水线（文件接力）**

```
orchestrator:
  claude -p "任务 A" -> results/a.json
  claude -p "读 results/a.json，做任务 B" -> results/b.json
  claude -p "读 results/b.json，做任务 C" -> results/c.json
  claude -p "读 results/*.json，综合"
```

没有并行，没有通讯问题。每步结果通过文件传递。

**模式 3：并行 fan-out + 文件 rendezvous（bash 编排）**

```bash
# 并行启动
claude -p "任务 A，结果写 results/a.json" &
claude -p "任务 B，结果写 results/b.json" &
claude -p "任务 C，结果写 results/c.json" &
wait  # 等待所有完成

# 综合
claude -p "读取 results/*.json，综合报告"
```

Agent 之间不通讯。通过 `wait` + 文件系统做同步点。

---

## 三层替代架构

### Tier 1：零基建 — AgentTool Background + Worktree

**适用**：单次 fan-out（一个 leader 分发 N 个独立任务，收集结果）。

```
Leader session
  |-- Agent(run_in_background: true, isolation: "worktree", prompt: "任务 A")
  |-- Agent(run_in_background: true, isolation: "worktree", prompt: "任务 B")
  +-- Agent(run_in_background: true, isolation: "worktree", prompt: "任务 C")
      |
  各 agent 在独立 worktree 中工作
      |
  完成后 <task-notification> 回注到 leader 的下一个 turn
      |
  Leader 综合结果
```

**优点**：零额外基建，Claude Code 原生支持。
**限制**：只有 parent<->child 通讯，agent 之间不能互相发消息。Parent session 退出则所有 agent 终止。并发 >5 可能 OOM。

### Tier 2：文件 IPC — 跨 session 协作

**适用**：需要跨 session 持久化、或 agent 数量超过单进程承载。

```
外部 orchestrator (bash/Python)
  |
  |-- 创建 .agent-coord/ 目录结构
  |     |-- tasks/          <-- JSON 任务文件（或 SQLite）
  |     |-- results/        <-- 命名输出文件
  |     +-- inbox/          <-- JSONL 格式消息队列
  |
  |-- 启动 worker: claude -p "读取 tasks/001.json，执行，结果写 results/001.json"
  |-- 启动 worker: claude -p "读取 tasks/002.json，执行，结果写 results/002.json"
  |
  |-- 等待完成（轮询 results/ 目录）
  |
  +-- 综合: claude -p "读取 results/*.json，综合报告"
```

**通讯协议**：

```json
// 任务文件 (.agent-coord/tasks/001.json):
{
  "id": "001",
  "status": "pending",
  "claimed_by": null,
  "claimed_at": null,
  "prompt": "修改 auth.ts ...",
  "depends_on": [],
  "result_path": null
}

// 结果文件 (.agent-coord/results/001.json):
{
  "task_id": "001",
  "status": "done",
  "summary": "修改了 auth.ts，增加了 token 刷新逻辑",
  "files_changed": ["src/auth.ts", "src/auth.test.ts"],
  "worktree_branch": "agent-001-auth-fix"
}
```

**任务认领**（防止并发冲突）：

```bash
# 方案 A：flock
flock -n .agent-coord/tasks/001.lock -c '
  status=$(jq -r .status .agent-coord/tasks/001.json)
  if [ "$status" = "pending" ]; then
    jq ".status=\"claimed\" | .claimed_by=\"$SESSION_ID\"" \
      .agent-coord/tasks/001.json > /tmp/t.json && mv /tmp/t.json .agent-coord/tasks/001.json
  fi
'

# 方案 B：SQLite（更可靠）
sqlite3 .agent-coord/tasks.db \
  "UPDATE tasks SET status='claimed', claimed_by='$SID' WHERE id=1 AND status='pending'"
```

**跨 session 状态共享**：

- 结果通过命名文件传递（`.agent-coord/results/{task-id}.json`）
- 不依赖 session transcript（agentId UUID 跨 session 不可追踪）
- Orchestrator 维护 `sessions.json`（session ID -> agent name 映射）

### Tier 3：完整编排器 — 长时间复杂流水线

**适用**：DAG 依赖、失败恢复、多日运行、可观测性需求。

```python
#!/usr/bin/env python3
# orchestrator.py — 最小可行编排器

import subprocess, json, time, sqlite3, os
from pathlib import Path

COORD_DIR = Path(".agent-coord")
DB = COORD_DIR / "tasks.db"

def init_db():
    db = sqlite3.connect(str(DB))
    db.execute("""CREATE TABLE IF NOT EXISTS tasks (
        id INTEGER PRIMARY KEY,
        prompt TEXT,
        status TEXT DEFAULT 'pending',
        depends_on TEXT DEFAULT '[]',
        claimed_by TEXT,
        result_path TEXT,
        attempts INTEGER DEFAULT 0,
        max_attempts INTEGER DEFAULT 3,
        created_at REAL,
        completed_at REAL
    )""")
    db.commit()
    return db

def claim_task(db, worker_id):
    """Lease-based claim: atomic UPDATE with WHERE status='pending'"""
    cur = db.execute("""
        UPDATE tasks SET status='claimed', claimed_by=?, attempts=attempts+1
        WHERE id = (
            SELECT id FROM tasks
            WHERE status='pending'
            AND NOT EXISTS (
                SELECT 1 FROM tasks AS dep
                WHERE dep.id IN (SELECT value FROM json_each(tasks.depends_on))
                AND dep.status != 'done'
            )
            ORDER BY id LIMIT 1
        )
        RETURNING id, prompt
    """, (worker_id,))
    row = cur.fetchone()
    db.commit()
    return row

def run_worker(task_id, prompt, worktree=True):
    """Spawn a claude -p worker, optionally in a git worktree"""
    result_path = COORD_DIR / f"results/{task_id}.json"

    wt_path = None
    if worktree:
        wt_path = f".claude/worktrees/task-{task_id}"
        subprocess.run(["git", "worktree", "add", wt_path, "-b", f"task-{task_id}"],
                       capture_output=True)

    cwd = wt_path or "."
    full_prompt = f"""{prompt}

完成后将结果写入 {result_path}，格式：
{{"task_id": "{task_id}", "status": "done", "summary": "...", "files_changed": [...]}}
"""

    proc = subprocess.run(
        ["claude", "-p", full_prompt],
        cwd=cwd, capture_output=True, text=True,
        timeout=600
    )
    return proc.returncode == 0

def orchestrate(db):
    """Main loop: claim tasks, spawn workers, collect results"""
    while True:
        pending = db.execute(
            "SELECT COUNT(*) FROM tasks WHERE status IN ('pending','claimed')"
        ).fetchone()[0]
        if pending == 0:
            break

        # Reclaim stale tasks (claimed > 15 min ago)
        db.execute("""
            UPDATE tasks SET status='pending', claimed_by=NULL
            WHERE status='claimed' AND claimed_by IS NOT NULL
            AND (julianday('now') - julianday(created_at)) * 86400 > 900
            AND attempts < max_attempts
        """)
        db.commit()

        task = claim_task(db, f"worker-{os.getpid()}")
        if task:
            task_id, prompt = task
            success = run_worker(task_id, prompt)
            status = "done" if success else "failed"
            db.execute(
                "UPDATE tasks SET status=?, completed_at=julianday('now') WHERE id=?",
                (status, task_id)
            )
            db.commit()
        else:
            time.sleep(5)

if __name__ == "__main__":
    db = init_db()
    orchestrate(db)
```

**可观测性**（hooks）：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{
          "type": "command",
          "command": "bash -c 'echo \"$(date): $TOOL_INPUT\" >> .agent-coord/audit.log'",
          "async": true
        }]
      }
    ]
  }
}
```

**失败恢复**：
- SQLite `attempts` + `max_attempts` 实现自动重试
- Stale lease 回收（claimed > 15 分钟未完成 -> 重置为 pending）
- Git worktree 残留检测 + 清理

**tmux 保活**：
```bash
tmux new-session -d -s orchestrator "cd /project && python orchestrator.py"
```

---

## 对比矩阵

| | Tier 1 AgentTool | Tier 2 文件 IPC | Tier 3 编排器 | Agent Teams (原生) |
|---|---|---|---|---|
| 基建成本 | 零 | 低（目录 + 脚本） | 中（Python + SQLite） | 零 |
| Peer 通讯 | 不支持 | 文件轮询（无保证） | 文件/SQLite | Mailbox（500ms） |
| Parent<->Child | 单向（notification only） | 双向（文件接力） | 双向（DB 状态） | 双向（SendMessage） |
| 跨 session | 不支持 | 支持 | 支持 | 支持 |
| 失败恢复 | 无 | 手动 | 自动（lease 回收） | 部分（resume） |
| 可观测性 | task-notification | 结果文件 | 审计日志 + DB 查询 | UI 面板 |
| 并发安全 | 同进程 | flock | SQLite WAL | proper-lockfile |
| 最大并发 | ~5（同进程 OOM） | ~10（进程隔离） | ~10（rate limit） | ~10（同进程） |
| 实时讨论/辩论 | **不可能** | **不可能** | **不可能** | 支持 |
| 适合场景 | 简单 fan-out | 中等复杂度 | 长时间流水线 | 交互式协作 |

---

## 推荐决策路径

```
需要多 agent 吗？
  +-- 是 -> agent 之间需要实时讨论/迭代吗？
  |         |-- 是 -> 必须用 Agent Teams（无替代）
  |         +-- 否 -> agent 之间需要通讯吗？
  |               |-- 否 -> Tier 1（AgentTool background + worktree）
  |               +-- 是 -> 需要跨 session 或长时间运行吗？
  |                     |-- 否 -> Tier 1 + 结果文件约定
  |                     +-- 是 -> 任务有依赖关系吗？
  |                           |-- 否 -> Tier 2（bash fan-out + 文件 IPC）
  |                           +-- 是 -> Tier 3（Python 编排器 + SQLite）
  +-- 否 -> 单 agent 就够
```

---

## 补充说明：`claude server` 不可用

Code-researcher 最终确认：`claude server` 受 `DIRECT_CONNECT` feature flag 保护，在外部构建中被死代码消除。Architecture C（hooks + server 的 WebSocket 注入方案）对外部用户不可行。

---

## Team Composition

| Agent | Role | Backend | Joined |
|-------|------|---------|--------|
| TL | Moderator (Session Lead) | Claude | Start |
| architect | Solution design, architecture | Claude | Round 1 |
| code-researcher | Source code verification | Claude | Round 1 |
| codex-proxy | OpenAI-family perspective | Codex (o4-mini) | Round 1 |
| gemini-proxy | Google-family perspective | Synthesized (quota exhausted) | Round 1 |

## Process Metadata
- Discussion rounds: 2
- Topics: 3 (all converged)
- Autonomous decisions: 3
- User escalations: 0
- Doodlestein: skipped (informational/advisory topics)

## Next Steps
- 如果要实现 Tier 2/3，可以 `/ae:plan` 规划具体的 orchestrator 脚本
- Tier 1 已经可以直接使用，无需额外开发
- 需要 agent 间实时讨论/辩论的场景，Agent Teams 没有替代方案
