---
id: "003-redis"
title: "Redis MCP IPC 可行性 + 动态调度 — Conclusion"
concluded: 2026-04-04
plan: ""
---

# Redis MCP IPC 可行性 + 动态调度 — Conclusion

## 关键发现（改变架构设计的事实）

| 发现 | 源码位置 | 影响 |
|------|---------|------|
| MCP tool 默认超时 **27.8 小时** | `mcp/client.ts:211` `DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000` | 阻塞式 SUBSCRIBE 会冻结 session 数小时 |
| Background agent **共享** parent MCP 连接 | `AgentTool/runAgent.ts:85-110` 返回 `parentClients` 同一引用 | SUBSCRIBE 阻塞会影响 parent 的 MCP 调用 |
| stdio MCP server 崩溃**不自动重连** | `useManageMCPConnections.ts:354` stdio 排除在 reconnect 之外 | Redis MCP 挂了需要手动恢复 |
| asyncRewake hook **无网络/进程限制** | `hooks.ts:205-245` spawn() 继承完整 env | 可以直接调 `redis-cli SUBSCRIBE` |
| Ctrl+C abort **传播到** MCP server | `mcp/client.ts:1426` 发 SIGTERM | 用户可以中断卡住的 MCP 调用 |

---

## Topic 3: 阻塞订阅 — converged

**Decision**: MCP tool 中**禁止**阻塞操作（SUBSCRIBE/BLPOP 无超时）。只用非阻塞模式。

**安全的 MCP tool 设计**:

| Tool | Redis 命令 | 阻塞 | 安全 |
|------|-----------|------|------|
| `queue_push` | `LPUSH` | 否 | OK |
| `queue_pop` | `RPOP` (非阻塞) | 否 | OK |
| `queue_pop_wait` | `BRPOP timeout=1` | 1 秒 | OK（可接受） |
| `queue_peek` | `LRANGE 0 0` | 否 | OK |
| `publish` | `PUBLISH` | 否 | OK |
| `subscribe` | `SUBSCRIBE` | **无限** | **禁止** |

**BRPOP timeout=1** 是可接受的折中——1 秒阻塞不会冻结 session，但如果模型在循环中反复调用会烧 token。

**额外保护**: 设置 `MCP_TOOL_TIMEOUT=10000`（10 秒）环境变量，防止任何 MCP tool 意外长时间阻塞。

**Rationale**: code-researcher 确认默认超时 27.8 小时，且 background agent 共享 parent MCP 连接。阻塞操作的风险不可接受。
**Reversibility**: high — 改 MCP tool 实现即可。

---

## Topic 4: Listener 模式 — converged

**Decision**: asyncRewake hook + redis-cli 是最佳 listener 模式。

**排名**:

| 方案 | Token 消耗 | 延迟 | 可靠性 | 推荐 |
|------|-----------|------|-------|------|
| asyncRewake + redis-cli SUBSCRIBE | 零（等待时） | <100ms | 中（一次性，需重启） | **首选** |
| 外部 daemon + 文件 inbox | 零（等待时） | ~1s | 高（独立进程） | 最可靠 |
| Background agent + queue_pop_wait 循环 | ~50K tok/hr @100msg | 1s | 中 | 低频场景 |
| MCP SUBSCRIBE | N/A | N/A | **session 冻结** | **禁止** |

**asyncRewake listener 实现**:

```bash
#!/bin/bash
# redis-listener.sh — asyncRewake hook
# 连接 Redis，等待一条消息，exit 2 唤醒模型

CHANNEL="${1:-ipc:default}"
TIMEOUT="${2:-300}"

# redis-cli SUBSCRIBE 输出格式: subscribe\nchannel\n1\nmessage\nchannel\npayload
MSG=$(timeout "$TIMEOUT" redis-cli SUBSCRIBE "$CHANNEL" 2>/dev/null | \
  awk '/^message$/{getline; getline; print; exit}')

if [ -n "$MSG" ]; then
  printf '{"hookSpecificOutput":{"additionalContext":"[Redis IPC] channel=%s\\n%s"}}\n' "$CHANNEL" "$MSG"
  exit 2  # 唤醒模型
fi

exit 0  # 超时，不唤醒
```

**hook 配置**:

```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "bash ~/.claude/scripts/redis-listener.sh ipc:session-a 300",
        "asyncRewake": true,
        "timeout": 310
      }]
    }]
  }
}
```

**重启问题**: asyncRewake 是一次性的——收到一条消息、唤醒模型后，hook 进程退出。需要重新启动。方案：
1. 模型处理完消息后，手动要求"继续监听"（触发下一个 cron turn 重启 hook）
2. Durable cron 每分钟检查 listener 是否存活，不存活就重启
3. 外部 daemon 模式不受此限制（daemon 持续运行，写文件 inbox 触发 asyncRewake）

**Rationale**: asyncRewake 是唯一不阻塞 session 且零 token 消耗的推送机制。redis-cli 是 Redis 自带的，不需要额外 MCP server。
**Reversibility**: high。

---

## Topic 5: Redis vs 文件 IPC — converged

**Decision**: 共存，按场景选择。不做静默 fallback。

| 场景 | 选择 | 理由 |
|------|------|------|
| 单 producer / consumer | 文件 IPC | 零依赖，已验证 |
| 消息频率 < 10/min | 文件 IPC | 轮询开销可忽略 |
| 多 worker 抢任务 | **Redis** | BRPOP 原子认领，文件锁不可靠 |
| Fan-out 到 N 个 session | **Redis** | PUBLISH 原生多播 |
| 需要 <100ms 延迟 | **Redis** | 文件轮询最快 1s |
| 零依赖环境 | 文件 IPC | 无需 Docker |

**适配器模式**:

```bash
# 通过环境变量切换后端
export IPC_BACKEND=file   # 或 redis
```

脚本内部根据 `IPC_BACKEND` 调用不同的 send/receive 实现。**不做静默 fallback**——Redis 不可用时直接报错，不偷偷切到文件，避免 split-brain。

**Rationale**: Architect 和 Codex 都确认 Redis 只在原子认领和多播场景有不可替代的优势。文件 IPC 已经验证通过，大部分场景够用。
**Reversibility**: high — 适配器可以随时加/换后端。

---

## Topic 6: 部署 — converged

**Decision**: Docker one-liner + uvx MCP server。

**最小安装**:

```bash
# 1. Redis（一条命令）
docker run -d --name claude-redis -p 6379:6379 --restart unless-stopped \
  redis:alpine redis-server --appendonly yes

# 2. MCP server（uvx 免安装）
# .mcp.json
{
  "mcpServers": {
    "redis-ipc": {
      "command": "uvx",
      "args": ["--from", "redis-ipc-mcp", "redis-ipc-mcp"]
    }
  }
}

# 3. 或者用 npx（Node 版）
{
  "mcpServers": {
    "redis-ipc": {
      "command": "npx",
      "args": ["-y", "redis-ipc-mcp"]
    }
  }
}
```

**Redis 挂了**: `--restart unless-stopped` 自动重启。stdio MCP server 不自动重连（源码确认），但 MCP server 本身不需要持久连接到 Redis——每次 tool call 建新连接即可。

**更轻量替代**:

| 方案 | 优点 | 缺点 | 推荐 |
|------|------|------|------|
| Redis (Docker) | 成熟、原子操作、pub/sub | Docker 依赖 | 多 worker 场景 |
| Named pipe (mkfifo) | 零依赖、macOS 原生、低延迟 | 只能 1:1、不持久、进程退出 pipe 断 | 两个 session 简单通讯 |
| SQLite WAL | 零依赖、ACID、丰富查询 | 无 pub/sub、只能轮询 | 任务队列（不需实时通知） |
| macOS XPC / distributed notifications | 系统原生 | 无可靠 payload、不跨用户 | 不推荐 |

**Rationale**: Docker + uvx 是最低摩擦的安装路径。AOF 保证消息持久化。
**Reversibility**: high — Docker 容器随时删。

---

## Topic 7: 动态任务分配 — converged

**Decision**: 中心化编排器 + queue-per-model + 混合伸缩。

### 架构

```
┌─ Orchestrator (Python) ────────────────────────────────────┐
│                                                             │
│  Task Intake → Classify → Route to queue:opus/haiku/sonnet  │
│                                                             │
│  Worker Manager:                                            │
│    ├─ min_warm_workers: {opus: 1, haiku: 2, sonnet: 1}     │
│    ├─ max_workers: {opus: 3, haiku: 5, sonnet: 3}          │
│    ├─ scale_up: queue_depth > 2 OR wait_p95 > 30s          │
│    ├─ scale_down: idle > 5min AND cooldown > 3min           │
│    └─ budget_governor: hourly_forecast > cap → freeze opus  │
│                                                             │
│  Health Monitor:                                            │
│    ├─ heartbeat: worker 每 15s 写 Redis key (TTL 30s)       │
│    ├─ stuck detection: no progress delta for 2x step time   │
│    └─ lease expiry: task.lease_until < now → requeue        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
         │                    │                    │
    queue:opus           queue:haiku          queue:sonnet
         │                    │                    │
    ┌────┴────┐          ┌────┴────┐          ┌────┴────┐
    │Worker A │          │Worker C │          │Worker E │
    │claude -p│          │claude -p│          │claude -p│
    │worktree │          │worktree │          │worktree │
    └─────────┘          └─────────┘          └─────────┘
    │Worker B │          │Worker D │
    │(elastic)│          │(elastic)│
    └─────────┘          └─────────┘
```

### Worker 生命周期

```
Orchestrator spawn → claude -p (in git worktree)
  → BRPOP queue:{model} timeout=30
    → 拿到任务 → 执行 → 写结果到 Redis + 文件
    → BRPOP 下一个任务（循环）
    → 空闲 > 5min → 自行退出
  → Orchestrator 检测到进程退出 → 清理 worktree
```

**Worker 实现**（claude -p 的 prompt 模板）：

```
你是一个 worker agent，从 Redis 队列领取任务并执行。

工作循环：
1. 调用 mcp__redis-ipc__queue_pop_wait(queue="queue:haiku", timeout=30)
2. 如果返回空 → 检查空闲时间，超过 5 分钟则退出
3. 如果返回任务 JSON → 解析并执行
4. 执行完成 → 写结果到 .agent-coord/results/{task_id}.json
5. 调用 mcp__redis-ipc__publish(channel="task:done", message='{task_id}')
6. 回到步骤 1

任务 JSON 格式：
{"task_id": "001", "prompt": "...", "result_path": "results/001.json"}
```

### 任务认领（BRPOP = 原子 tryClaimNextTask）

```
Agent Teams:
  tryClaimNextTask() → 文件锁 + UPDATE owner → 10+ 并发 agent 竞争

Redis:
  BRPOP queue:haiku 30 → 第一个 popper 拿到任务，其他等下一个
  
  原子性保证：Redis 单线程，BRPOP 不会双发。
  比 Agent Teams 的文件锁方案更可靠。
```

### 弹性伸缩

```python
# 伸缩逻辑（orchestrator 每 30s 执行）
for model in ['opus', 'haiku', 'sonnet']:
    depth = redis.llen(f'queue:{model}')
    active = count_active_workers(model)
    
    # Scale up
    if depth > 2 and active < max_workers[model]:
        spawn_worker(model)
    
    # Scale down (with hysteresis)
    if depth == 0 and idle_time(model) > 300 and active > min_warm[model]:
        if time_since_last_scale > 180:  # 3min cooldown
            terminate_idle_worker(model)
    
    # Budget governor
    if forecast_hourly_cost() > budget_cap:
        freeze_scaleup('opus')
        downgrade_eligible_tasks('opus', 'sonnet')
```

### 故障恢复

| 故障 | 检测 | 恢复 |
|------|------|------|
| Worker 崩溃 | 进程退出码非 0 | 任务回到队列（BRPOP 未 ACK = 已丢失，需要用 RPOPLPUSH 模式） |
| Worker 卡住 | Heartbeat TTL 过期 | Orchestrator kill -9 + requeue |
| Redis 挂了 | 连接失败 | Docker --restart 自动重启，worker 重试连接 |
| Orchestrator 挂了 | systemd/launchd 监控 | 自动重启，worker 继续 BRPOP 不受影响 |

**可靠消息模式**（防止 worker 崩溃丢任务）：

```
# 用 RPOPLPUSH 替代 BRPOP
RPOPLPUSH queue:haiku processing:worker-a
# worker 完成后
LREM processing:worker-a 1 <task>
# worker 崩溃后，orchestrator 将 processing:worker-a 的任务移回 queue
```

---

## 完整推荐架构

```
层级 1 — 消息传递
  ├── 简单场景: 文件 IPC（session-ipc-*.sh，已验证）
  └── 多 worker 场景: Redis（LPUSH/BRPOP 队列 + PUBLISH 通知）

层级 2 — 消息监听
  ├── asyncRewake hook + redis-cli SUBSCRIBE（零 token，推送唤醒）
  ├── 外部 daemon + 文件 inbox（最可靠）
  └── Durable cron 轮询（保底，零依赖）

层级 3 — 任务调度
  ├── 简单: bash fan-out（parallel claude -p + wait）
  ├── 中等: Redis queue + pre-spawn workers
  └── 完整: Python orchestrator + queue-per-model + 弹性伸缩 + 故障恢复

层级 4 — 文件隔离
  └── Git worktree per worker（无条件可用，无 feature gate）
```

**注意：MCP tool 中绝对不要用阻塞式 SUBSCRIBE。** 用 asyncRewake hook + redis-cli 做监听，MCP tool 只做非阻塞的队列操作。

---

## Team Composition

| Agent | Role | Backend | Joined |
|-------|------|---------|--------|
| TL | Moderator | Claude | Start |
| architect | Architecture design | Claude | Round 1 |
| code-researcher | Source code verification | Claude | Round 1 |
| codex-proxy | Orchestration patterns | Codex (o4-mini) | Round 1 |

## Process Metadata
- Discussion rounds: 1 (strong convergence, no revisit needed)
- Topics: 5 (all converged)
- Autonomous decisions: 5
- User escalations: 0
- Critical finding: MCP tool timeout 27.8h + shared MCP connection → changed entire architecture

## Next Steps
- `/ae:plan` 如果要实现 Redis MCP server + orchestrator
- 文件 IPC 已经可用，Redis 是增量升级
- asyncRewake + redis-cli listener 可以立即测试（不需要 MCP server）
