# Claude Code 跨 Session IPC 方案

不修改源代码，利用现有机制实现多个 Claude Code session 之间的通讯。

---

## 原理

Claude Code 没有内置的跨 session 通讯机制（SendMessage/Mailbox 属于 Agent Teams）。但源码中有以下可利用的原语：

| 原语 | 源码位置 | 能力 |
|------|---------|------|
| `asyncRewake` hook | `hooks.ts:205-208` | 后台运行，exit 2 时主动唤醒模型 |
| `FileChanged` hook | `fileChangedWatcher.ts` | 检测外部文件变化 |
| `additionalContext` 注入 | `hooks.ts:347,434` | hook 输出 JSON 可注入 system-reminder |
| `watchPaths` 动态注册 | `hooks.ts:353` | CwdChanged/FileChanged hook 可动态添加监听路径 |
| 文件系统 | Read/Write/Bash | 任何 session 都可读写 |
| `flock` | 系统调用 | 文件锁防止并发写入 |

**核心发现**：`asyncRewake: true` 是源码中唯一的"推送唤醒"机制——hook 后台运行，检测到事件后 `exit 2`，Claude Code 将其注入为 task-notification 唤醒模型。不需要 Agent Teams。

---

## 目录结构

```
{project}/
  .agent-coord/
    sessions.json                 # session 注册表 {"name": "session-id"}
    sessions.lock                 # 注册表写锁
    inbox-{session-name}/         # 每个 session 的收件箱
      msg-{timestamp}-{pid}.jsonl # 消息文件
      .processed/                 # 已处理的消息
      .last_check                 # 最后检查时间戳
      .lock                       # 写入锁
```

## 消息格式

```json
{
  "from": "session-a",
  "to": "session-b",
  "timestamp": "2026-04-04T12:00:00Z",
  "type": "result",
  "text": "任务 A 完成，结果在 results/a.json",
  "data_path": "results/a.json"
}
```

`type` 值：`result`（任务结果）、`request`（请求执行）、`status`（状态更新）、`command`（指令）。

---

## 脚本工具集

所有脚本安装在 `~/.claude/scripts/`。

### session-ipc-init.sh — 初始化 IPC

```bash
#!/bin/bash
# 用法: bash session-ipc-init.sh <coord-dir> <session-name>
# 创建 inbox 目录，注册到 sessions.json

COORD="${1:-.agent-coord}"
SESSION_NAME="${2:-$(echo $RANDOM | md5sum | head -c 8)}"

mkdir -p "$COORD/inbox-$SESSION_NAME" "$COORD/inbox-$SESSION_NAME/.processed"

if [ ! -f "$COORD/sessions.json" ]; then
  echo "{}" > "$COORD/sessions.json"
fi

flock "$COORD/sessions.lock" bash -c "
  jq --arg name \"$SESSION_NAME\" --arg sid \"${CLAUDE_SESSION_ID:-unknown}\" \
    '.[\$name] = \$sid' \"$COORD/sessions.json\" > \"$COORD/sessions.json.tmp\" \
    && mv \"$COORD/sessions.json.tmp\" \"$COORD/sessions.json\"
"

echo "$SESSION_NAME"
```

### session-ipc-send.sh — 发送消息

```bash
#!/bin/bash
# 用法: bash session-ipc-send.sh <coord-dir> <from> <to> <message> [data_path]

set -euo pipefail

COORD="${1:-.agent-coord}"
FROM="$2"
TO="$3"
TEXT="$4"
DATA_PATH="${5:-}"

TARGET_INBOX="$COORD/inbox-$TO"

if [ ! -d "$TARGET_INBOX" ]; then
  echo "error: target session '$TO' inbox not found" >&2
  exit 1
fi

TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
MSGID="msg-$(date +%s)-$$"

flock "$TARGET_INBOX/.lock" bash -c "
  cat > \"$TARGET_INBOX/${MSGID}.jsonl\" <<'MSGEOF'
{\"from\":\"$FROM\",\"to\":\"$TO\",\"timestamp\":\"$TIMESTAMP\",\"text\":\"$TEXT\",\"data_path\":\"$DATA_PATH\"}
MSGEOF
"

echo "sent: $FROM -> $TO ($MSGID)"
```

### session-ipc-watch.sh — 监听收件箱（asyncRewake 用）

```bash
#!/bin/bash
# 用法: bash session-ipc-watch.sh <coord-dir> <session-name> [poll_interval] [max_wait]
# Exit 0 = 无消息（不唤醒）
# Exit 2 = 有新消息（唤醒模型）

set -euo pipefail

COORD="${1:-.agent-coord}"
SESSION_NAME="${2:-default}"
INBOX="$COORD/inbox-$SESSION_NAME"
POLL_INTERVAL="${3:-1}"
MAX_WAIT="${4:-300}"

[ -d "$INBOX" ] || exit 0

ELAPSED=0

while [ "$ELAPSED" -lt "$MAX_WAIT" ]; do
  # 检查未处理的 .jsonl
  if [ -f "$INBOX/.last_check" ]; then
    UNREAD=$(find "$INBOX" -maxdepth 1 -name "*.jsonl" -newer "$INBOX/.last_check" 2>/dev/null | head -5)
  else
    UNREAD=$(find "$INBOX" -maxdepth 1 -name "*.jsonl" 2>/dev/null | head -5)
  fi

  if [ -n "$UNREAD" ]; then
    MESSAGES=""
    for f in $UNREAD; do
      MSG=$(cat "$f" 2>/dev/null || true)
      [ -n "$MSG" ] && MESSAGES="${MESSAGES}${MSG}\n"
      mv "$f" "$INBOX/.processed/" 2>/dev/null || true
    done

    touch "$INBOX/.last_check"

    CLEAN_MSG=$(echo -e "$MESSAGES" | head -c 2000)
    cat <<HOOKEOF
{"hookSpecificOutput":{"additionalContext":"[IPC Message]\n${CLEAN_MSG}"}}
HOOKEOF
    exit 2  # 唤醒模型
  fi

  sleep "$POLL_INTERVAL"
  ELAPSED=$((ELAPSED + POLL_INTERVAL))
done

exit 0
```

### session-ipc-list.sh — 查看状态

```bash
#!/bin/bash
# 用法: bash session-ipc-list.sh <coord-dir>

COORD="${1:-.agent-coord}"

echo "=== Sessions ==="
if [ -f "$COORD/sessions.json" ]; then
  jq -r 'to_entries[] | "  \(.key) -> \(.value)"' "$COORD/sessions.json" 2>/dev/null || echo "  (none)"
fi

echo ""
echo "=== Inboxes ==="
for inbox in "$COORD"/inbox-*/; do
  [ -d "$inbox" ] || continue
  NAME=$(basename "$inbox" | sed 's/^inbox-//')
  UNREAD=$(find "$inbox" -maxdepth 1 -name "*.jsonl" 2>/dev/null | wc -l | tr -d ' ')
  PROCESSED=$(find "$inbox/.processed" -maxdepth 1 -name "*.jsonl" 2>/dev/null | wc -l | tr -d ' ')
  echo "  $NAME: $UNREAD unread, $PROCESSED processed"
done
```

---

## 三种使用模式

### 模式 1：手动收发（最简单）

两个 tmux pane 各跑一个 Claude Code session，手动收发消息。

**Session A**：
```bash
cd /project

# 初始化
! bash ~/.claude/scripts/session-ipc-init.sh .agent-coord session-a

# 告诉模型身份
> 我是 session-a。要发消息用：
> bash ~/.claude/scripts/session-ipc-send.sh .agent-coord session-a session-b "内容"
> 要收消息用：cat .agent-coord/inbox-session-a/*.jsonl
```

**Session B**：
```bash
cd /project
! bash ~/.claude/scripts/session-ipc-init.sh .agent-coord session-b
# 同上
```

**发消息**（Session A 中让模型执行）：
```bash
bash ~/.claude/scripts/session-ipc-send.sh .agent-coord session-a session-b "auth 模块分析完了，发现3个问题，详见 results/auth-analysis.json"
```

**收消息**（Session B 中手动或让模型检查）：
```bash
cat .agent-coord/inbox-session-b/*.jsonl
```

### 模式 2：asyncRewake 自动唤醒（推送模式）

Session 启动时自动开始监听收件箱，有新消息时模型被唤醒。

**项目级 hook 配置**（`.claude/settings.json`）：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [{
          "type": "command",
          "command": "bash /Users/ckai/.claude/scripts/session-ipc-watch.sh .agent-coord ${CLAUDE_SESSION_NAME:-default} 1 300",
          "asyncRewake": true,
          "timeout": 310
        }]
      }
    ]
  }
}
```

**启动时设置 session name**：

```bash
CLAUDE_SESSION_NAME=session-a claude
```

**工作流**：

```
Session A 启动 -> asyncRewake watch 后台开始监听
  |
Session B 发消息 -> 写入 inbox-session-a/
  |
watch 检测到新文件 -> exit 2
  |
Claude Code 注入 task-notification
  |
Session A 模型被唤醒:
  "[IPC Message]
   {"from":"session-b","text":"需要你分析一下 auth.ts"}"
  |
Session A 模型自动处理消息
```

**注意**：
- asyncRewake 是源码中有的机制（`hooks.ts:205`），但需要验证当前版本是否支持
- watch 脚本每次唤醒后退出，持续监听需要在 SessionStart hook 中加 loop，或接受每次唤醒后需要手动/cron 重启 watch
- 如果 asyncRewake 不可用，退化为模式 1（手动收发），不会报错

### 模式 3：Cron 轮询（零依赖保底）

不依赖 asyncRewake，用 durable cron 定期检查收件箱。

**在每个 session 中设置**：

```
创建 durable recurring cron（每 1 分钟）：
检查 .agent-coord/inbox-${SESSION_NAME}/ 有没有 *.jsonl 文件。
如果有，读取内容，处理消息，处理完后移动到 .processed/ 目录。
如果没有，什么也不做（不要输出任何内容）。
```

**优点**：纯 Claude Code 内置功能，零外部依赖。
**缺点**：1 分钟延迟，每次轮询消耗 turn token（即使没消息）。

---

## 完整工作示例：两个 Session 协作分析代码

### 目标

Session A 分析前端代码，Session B 分析后端代码，互相交换发现，最终综合。

### 步骤

**1. 初始化（两个 terminal 都执行）**

```bash
# Terminal 1
cd /project
CLAUDE_SESSION_NAME=frontend claude --name frontend

# 在 Claude 中:
> ! bash ~/.claude/scripts/session-ipc-init.sh .agent-coord frontend

# Terminal 2
cd /project
CLAUDE_SESSION_NAME=backend claude --name backend

# 在 Claude 中:
> ! bash ~/.claude/scripts/session-ipc-init.sh .agent-coord backend
```

**2. 各自工作**

Terminal 1（frontend session）：
```
分析 src/components/ 和 src/hooks/ 中的前端代码，
重点看状态管理和性能问题。
分析完后把发现写到 .agent-coord/results/frontend.json，
然后发消息通知 backend session：
bash ~/.claude/scripts/session-ipc-send.sh .agent-coord frontend backend "前端分析完成，结果在 results/frontend.json"
```

Terminal 2（backend session）：
```
分析 src/services/ 和 src/tools/ 中的后端代码，
重点看 API 设计和错误处理。
分析完后把发现写到 .agent-coord/results/backend.json，
然后发消息通知 frontend session：
bash ~/.claude/scripts/session-ipc-send.sh .agent-coord backend frontend "后端分析完成，结果在 results/backend.json"
```

**3. 交换发现**

各自收到对方的消息后：
```
# frontend session
读取 .agent-coord/inbox-frontend/ 中的消息，
然后读取 .agent-coord/results/backend.json，
结合前端的发现，找出前后端交互的问题。
```

```
# backend session
同样读取对方的结果，从后端角度补充分析。
```

**4. 综合（任选一个 session）**

```
读取 .agent-coord/results/frontend.json 和 backend.json，
综合两方的分析，写出最终报告到 .agent-coord/results/final-report.md
```

---

## 外部编排器版本

如果需要自动化（不手动操作两个 terminal），用 bash 脚本编排：

```bash
#!/bin/bash
# orchestrate.sh — 两个 session 并行分析 + 综合
set -euo pipefail

PROJECT="/path/to/project"
COORD="$PROJECT/.agent-coord"

# 初始化
bash ~/.claude/scripts/session-ipc-init.sh "$COORD" frontend
bash ~/.claude/scripts/session-ipc-init.sh "$COORD" backend
mkdir -p "$COORD/results"

# 并行启动两个 headless session
cd "$PROJECT"

claude -p "分析 src/components/ 和 src/hooks/ 的前端代码。
重点：状态管理、性能。
结果写到 .agent-coord/results/frontend.json（JSON 格式，包含 findings 数组）。" &
PID_A=$!

claude -p "分析 src/services/ 和 src/tools/ 的后端代码。
重点：API 设计、错误处理。
结果写到 .agent-coord/results/backend.json（JSON 格式，包含 findings 数组）。" &
PID_B=$!

# 等待完成
wait $PID_A $PID_B

# 综合
claude -p "读取 .agent-coord/results/frontend.json 和 backend.json，
综合前后端分析，写出最终报告到 .agent-coord/results/final-report.md。
重点关注前后端交互的问题。"

echo "Done: $COORD/results/final-report.md"
```

```bash
# 运行
bash orchestrate.sh
```

---

## 方案 D：Redis MCP Server（Docker + pub/sub）

文件 IPC 的根本问题是没有推送机制——只能轮询或依赖 asyncRewake。Redis pub/sub 天然解决这个问题。

### 架构

```
Session A                               Session B
  |                                       |
  MCP tool: redis_publish                 MCP tool: redis_subscribe (blocking)
  channel: "to-session-b"                 channel: "to-session-b"
  |                                       |
  +---------> Redis (localhost:6379) <----+
              docker container
```

### 优势

| | 文件 IPC | Redis MCP |
|---|---|---|
| 延迟 | 1s 轮询 | <1ms（内存 pub/sub） |
| 推送 | 无（轮询或 asyncRewake） | 原生（SUBSCRIBE 阻塞等待） |
| 多播 | 需要逐个写 inbox | 原生（一个 channel 多个 subscriber） |
| 并发安全 | mkdir 锁 / mktemp+mv | Redis 单线程，无锁问题 |
| 可靠队列 | 文件 + .processed 目录 | LPUSH/BRPOP（阻塞弹出） |
| 消息持久化 | 文件天然持久 | 需要 Redis AOF 或 Stream |

### 启动 Redis

```bash
docker run -d --name claude-redis -p 6379:6379 redis:alpine
```

### MCP Server 设计

一个最小 Redis MCP server 只需要 4 个 tool：

| Tool | 作用 | Redis 命令 |
|------|------|-----------|
| `redis_publish` | 发送消息到 channel | `PUBLISH channel message` |
| `redis_subscribe_once` | 阻塞等待一条消息 | `SUBSCRIBE channel` → 收到一条后返回 |
| `redis_queue_push` | 推入可靠队列 | `LPUSH queue message` |
| `redis_queue_pop` | 阻塞弹出（带超时） | `BRPOP queue timeout` |

pub/sub 用于实时通知，queue（List）用于可靠消息传递（不丢消息）。

### MCP Server 实现（Python，最小可行）

```python
#!/usr/bin/env python3
"""redis-ipc-mcp — Redis-based IPC for Claude Code sessions"""

import json
import redis
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("redis-ipc")
pool = redis.ConnectionPool(host="localhost", port=6379, decode_responses=True)

def get_redis():
    return redis.Redis(connection_pool=pool)

@mcp.tool()
def publish(channel: str, message: str) -> str:
    """Publish a message to a Redis channel. All subscribers receive it instantly."""
    r = get_redis()
    count = r.publish(channel, message)
    return f"published to {channel} ({count} subscribers)"

@mcp.tool()
def subscribe_once(channel: str, timeout: int = 30) -> str:
    """Block until one message arrives on the channel, or timeout.
    Use this in a background agent to wait for messages from other sessions."""
    r = get_redis()
    ps = r.pubsub()
    ps.subscribe(channel)
    # Skip the subscription confirmation message
    ps.get_message(timeout=1)
    # Wait for actual message
    msg = ps.get_message(timeout=timeout)
    ps.unsubscribe()
    ps.close()
    if msg and msg["type"] == "message":
        return msg["data"]
    return ""  # timeout, no message

@mcp.tool()
def queue_push(queue: str, message: str) -> str:
    """Push a message to a reliable queue (Redis List). Messages persist until consumed."""
    r = get_redis()
    length = r.lpush(queue, message)
    return f"queued in {queue} (depth: {length})"

@mcp.tool()
def queue_pop(queue: str, timeout: int = 10) -> str:
    """Pop a message from a reliable queue. Blocks up to timeout seconds.
    Returns empty string if timeout. Messages are removed on pop (at-most-once)."""
    r = get_redis()
    result = r.brpop(queue, timeout=timeout)
    if result:
        return result[1]
    return ""  # timeout

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

**依赖**：`pip install mcp redis`

### 注册 MCP Server

```json
// .mcp.json（项目级）
{
  "mcpServers": {
    "redis-ipc": {
      "command": "python3",
      "args": ["/path/to/redis_ipc_mcp.py"]
    }
  }
}
```

或用 uvx 免安装：

```json
{
  "mcpServers": {
    "redis-ipc": {
      "command": "uvx",
      "args": ["--from", "git+https://your-repo/redis-ipc-mcp", "redis-ipc-mcp"]
    }
  }
}
```

### 使用模式

**模式 A：直接 pub/sub（最低延迟）**

Session B 启动一个 background agent 做 subscriber：

```
spawn 一个 background agent，持续监听 Redis channel "to-session-b"：
1. 调用 mcp__redis-ipc__subscribe_once(channel="to-session-b", timeout=60)
2. 如果收到消息，处理它
3. 处理完后再次调用 subscribe_once 继续监听
4. 如果 timeout 无消息，继续循环
```

Session A 发消息：

```
调用 mcp__redis-ipc__publish(channel="to-session-b", message='{"from":"a","text":"auth 分析完了"}')
```

**问题**：`subscribe_once` 是阻塞的 MCP tool call。Background agent 在等待期间那个 turn 不会结束。如果用 `run_in_background: true`，agent 会在后台阻塞等待，不影响主 session。完成后通过 `<task-notification>` 回报。但每次只能收一条消息就需要一个新 agent。

**模式 B：可靠队列（推荐）**

不用 pub/sub 的阻塞订阅，改用 List 队列 + 定期检查：

Session A 发消息：
```
mcp__redis-ipc__queue_push(queue="inbox:session-b", message='{"from":"a","text":"分析完了","data":"results/a.json"}')
```

Session B 检查消息（通过 cron 或手动）：
```
mcp__redis-ipc__queue_pop(queue="inbox:session-b", timeout=1)
```

`queue_pop` 超时 1 秒——有消息立刻返回，无消息 1 秒后返回空。不会长时间阻塞。

配合 `/loop`：
```
/loop 1m 检查 Redis 队列：调用 mcp__redis-ipc__queue_pop(queue="inbox:session-b", timeout=1)，如果有消息就处理
```

**模式 C：Stream（最完整，需要更多 tool）**

Redis Stream 提供消费者组、消息确认、历史回溯。适合生产级场景，但 MCP server 需要更多 tool（`XADD`、`XREADGROUP`、`XACK`）。可以作为后续扩展。

### 对比

| | 文件 IPC（方案 A-C） | Redis MCP（方案 D） |
|---|---|---|
| 基建 | 零 | Docker + Python MCP server |
| 延迟 | 1s 轮询 / asyncRewake | <1ms pub/sub / 1s queue poll |
| 可靠性 | 原子 mv + mkdir 锁 | Redis 单线程 + AOF |
| 多播 | 逐个写 inbox | pub/sub 原生 |
| 持久化 | 文件天然持久 | 需配置 AOF/RDB |
| 可观测 | `session-ipc-list.sh` | `redis-cli LLEN inbox:xxx` |
| 复杂度 | 低 | 中（需 Docker + MCP server） |
| 适合 | 简单场景，零依赖 | 多 session 高频通讯 |

### 现成方案

如果不想自己写 MCP server，可以找现有的 Redis MCP server（社区有多个实现），只要支持 `PUBLISH`/`SUBSCRIBE`/`LPUSH`/`BRPOP` 就够用。关键是 `BRPOP` 的阻塞语义——确认 MCP server 实现了带 timeout 的阻塞弹出。

---

## 注意事项

### 并发安全

- 所有写操作通过 `flock` 保护——多个 session 同时写同一个 inbox 不会损坏文件
- 消息文件名包含 timestamp + PID，保证唯一性
- `sessions.json` 更新也有 `flock` 保护

### Rate Limit

- 并行 `claude -p` 调用共享同一个 API key 的 rate limit
- 建议并行数不超过 3-5 个
- 如果遇到 429，加 `sleep` 错开启动时间：
  ```bash
  claude -p "任务 A" &
  sleep 2
  claude -p "任务 B" &
  sleep 2
  claude -p "任务 C" &
  wait
  ```

### 清理

```bash
# 清理所有 IPC 状态
rm -rf .agent-coord/

# 只清理已处理的消息
find .agent-coord/ -path "*/.processed/*" -delete
```

### .gitignore

```
# 添加到项目 .gitignore
.agent-coord/
```

### asyncRewake 的限制

- 每次唤醒后 watch 脚本退出——不是持续监听
- 需要重新启动 watch（通过 cron 或模型手动重启）
- 如果当前版本不支持 asyncRewake，hook 正常运行但 exit 2 不会唤醒模型
- 验证方式：在 settings.json 配置一个简单的 asyncRewake hook，检查 exit 2 是否触发 task-notification

### 全方案对比

| 能力 | 文件 IPC (A-C) | Redis MCP (D) | Agent Teams (原生) |
|------|---------------|---------------|-------------------|
| 消息延迟 | 1s 轮询 / asyncRewake ~100ms | <1ms pub/sub | <1ms 内存 / 500ms mailbox |
| 自动唤醒 | asyncRewake（需验证） | background agent + notification | 原生 500ms 轮询 |
| Peer-to-Peer | 通过文件 | pub/sub channel | SendMessage 直接寻址 |
| 多播 | 逐个写 inbox | pub/sub 原生 | `to: "*"` broadcast |
| 模型感知 | 需要 prompt 告知协议 | MCP tool 原生可调用 | 原生 `<teammate-message>` |
| 并发安全 | mkdir 锁 / mktemp+mv | Redis 单线程 | proper-lockfile |
| 持久化 | 文件天然持久 | 需配 AOF | 文件系统 |
| 外部依赖 | 无 | Docker + Python | 无 |
| 生命周期管理 | 无 | 无 | idle/wake + shutdown |
| 适合场景 | 简单/零依赖 | 高频多 session | 交互式协作 |
