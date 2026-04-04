---
id: "003"
title: "跨 Session 通讯方案 — Conclusion"
concluded: 2026-04-04
plan: ""
---

# 跨 Session 通讯方案 — Conclusion

基于源码中已有的机制，不修改代码，实现多个 Claude Code session 之间通讯。

---

## 可用的 IPC 原语（源码验证）

### 原语 1：FileChanged Hook + watchPaths

**源码**：`src/utils/hooks/fileChangedWatcher.ts`, `hooksConfigManager.ts:259-262`

配置一个 `FileChanged` hook，监听特定文件。当文件被外部进程（另一个 session）修改时，hook 触发。

```json
// settings.json
{
  "hooks": {
    "FileChanged": [
      {
        "matcher": ".agent-coord/inbox.jsonl",
        "hooks": [{
          "type": "command",
          "command": "cat .agent-coord/inbox.jsonl | tail -1"
        }]
      }
    ]
  }
}
```

**能力**：检测外部文件变化。
**限制**：hook 输出可以注入 `additionalContext` 但不能直接唤醒模型——模型只在下一个 turn 开始时看到。

### 原语 2：asyncRewake Hook

**源码**：`src/utils/hooks.ts:205-208`

```
asyncRewake hooks bypass the registry entirely. On completion, if exit
code 2 (blocking error), enqueue as a task-notification so it wakes the
model via useQueueProcessor (idle) or gets injected mid-query via
queued_command attachments (busy).
```

**关键发现**：`asyncRewake: true` 的 hook 在后台运行，退出码为 2 时**主动唤醒模型**——注入为 task-notification。这是唯一不依赖 Agent Teams 的"推送"机制。

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [{
          "type": "command",
          "command": "bash ~/.claude/scripts/watch-inbox.sh",
          "asyncRewake": true,
          "timeout": 3600
        }]
      }
    ]
  }
}
```

`watch-inbox.sh` 可以 `inotifywait` 监听文件变化，有新消息时 `exit 2` 唤醒模型。

### 原语 3：additionalContext 注入

**源码**：`src/utils/hooks.ts:347,434-438`

Hook 的 JSON 输出中可以包含 `hookSpecificOutput.additionalContext`。这段文本会作为 system-reminder 注入到模型的下一个 turn。

```bash
#!/bin/bash
# Hook 输出 JSON，注入 context
MESSAGE=$(cat .agent-coord/inbox.jsonl | tail -1)
echo "{\"hookSpecificOutput\":{\"additionalContext\":\"收到来自 Session B 的消息: $MESSAGE\"}}"
```

**能力**：将外部信息注入到当前 session 的模型 context。
**限制**：只在 hook 触发时注入，不是持续可见的。

### 原语 4：文件系统读写

最基础的原语。任何 session 都可以读写文件。通过约定文件路径实现消息传递。

---

## 三个通讯方案

### 方案 A：文件信号 + FileChanged + asyncRewake（最可行）

**架构**：

```
Session A                           Session B
  |                                   |
  | 写文件:                           |
  | .agent-coord/to-b/msg-001.jsonl   |
  |                                   |
  |                         FileChanged hook 检测到文件变化
  |                         asyncRewake watch 脚本 exit 2
  |                                   |
  |                         模型被唤醒，注入 additionalContext:
  |                         "Session A 发来消息: ..."
  |                                   |
  |                         模型读取消息，处理，写回复:
  |                         .agent-coord/to-a/msg-001.jsonl
  |                                   |
  | FileChanged hook 检测到           |
  | asyncRewake exit 2 唤醒模型       |
  | 读取回复                           |
```

**设置**：

```json
// settings.json — 两个 session 都需要配置
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [{
          "type": "command",
          "command": "bash ~/.claude/scripts/watch-inbox.sh",
          "asyncRewake": true,
          "timeout": 3600
        }]
      }
    ]
  }
}
```

**watch-inbox.sh**：

```bash
#!/bin/bash
# 监听自己的收件箱目录
INBOX=".agent-coord/to-${CLAUDE_SESSION_NAME:-default}"
mkdir -p "$INBOX"

# macOS: 用 fswatch; Linux: 用 inotifywait
if command -v fswatch &>/dev/null; then
  fswatch -1 "$INBOX" >/dev/null 2>&1
elif command -v inotifywait &>/dev/null; then
  inotifywait -q -e create,modify "$INBOX"
fi

# 有新文件 -> exit 2 唤醒模型
LATEST=$(ls -t "$INBOX"/*.jsonl 2>/dev/null | head -1)
if [ -n "$LATEST" ]; then
  MSG=$(cat "$LATEST")
  echo "{\"hookSpecificOutput\":{\"additionalContext\":\"收到跨 session 消息: $MSG\"}}"
  exit 2  # 唤醒模型
fi
exit 0  # 无消息，不唤醒
```

**发送消息**（在 session 内用 Bash 工具）：

```bash
# Session A 发消息给 Session B
echo '{"from":"session-a","text":"任务 A 完成，结果在 results/a.json"}' \
  > .agent-coord/to-session-b/msg-$(date +%s).jsonl
```

**优点**：
- 真正的推送唤醒（asyncRewake exit 2）
- 不需要轮询（fswatch/inotifywait 是 OS 级事件）
- additionalContext 让模型直接看到消息内容

**限制**：
- asyncRewake 是否在所有外部构建中可用需要验证
- fswatch 需要额外安装（`brew install fswatch`）
- watch 脚本是一次性的——唤醒一次后需要重新启动监听（可以用 loop 处理）

### 方案 B：外部 Watchdog 编排器

**架构**：

```
Watchdog 脚本（独立进程，持续运行）
  |
  |-- 监听 .agent-coord/outbox/ 目录
  |-- 检测新消息文件
  |-- 路由: 读取 to 字段，写入目标 session 的 inbox
  |-- 唤醒目标: claude -p --resume <session-id> "检查 inbox"
  |
Session A <--> .agent-coord/outbox/  <--> Watchdog <--> .agent-coord/inbox-b/ <--> Session B
```

**watchdog.sh**：

```bash
#!/bin/bash
# watchdog.sh — 跨 session 消息路由器
COORD=".agent-coord"
SESSIONS="$COORD/sessions.json"  # {"session-a": "uuid-xxx", "session-b": "uuid-yyy"}

mkdir -p "$COORD/outbox"

fswatch -0 "$COORD/outbox" | while read -d '' file; do
  [ -f "$file" ] || continue

  TO=$(jq -r '.to' "$file")
  FROM=$(jq -r '.from' "$file")
  TEXT=$(jq -r '.text' "$file")

  # 写入目标 inbox
  TARGET_INBOX="$COORD/inbox-$TO"
  mkdir -p "$TARGET_INBOX"
  cp "$file" "$TARGET_INBOX/"

  # 如果目标是 headless session，唤醒它
  SESSION_ID=$(jq -r ".[\"$TO\"]" "$SESSIONS")
  if [ "$SESSION_ID" != "null" ]; then
    claude -p --resume "$SESSION_ID" \
      "收到来自 $FROM 的消息: $TEXT。读取 $TARGET_INBOX/ 中的完整消息并处理。" &
  fi

  # 归档已路由的消息
  mv "$file" "$COORD/outbox/.archived/"
done
```

**优点**：
- 完全在 Claude Code 外部，不依赖任何内部机制
- 可以路由给 headless session（通过 `claude -p --resume` 唤醒）
- 消息有归档，可追溯

**限制**：
- 需要一个持续运行的 watchdog 进程
- `claude -p --resume` 唤醒有冷启动开销（1-2s）
- 每次唤醒都是一个新 turn，消耗 token

### 方案 C：Durable Cron 轮询（零外部依赖）

**架构**：

```
Session A                               Session B
  |                                       |
  | Durable cron (每 1 分钟)              | Durable cron (每 1 分钟)
  |   -> 检查 .agent-coord/inbox-a/      |   -> 检查 .agent-coord/inbox-b/
  |   -> 有新消息 -> 处理                  |   -> 有新消息 -> 处理
  |   -> 写回复到 inbox-b/               |   -> 写回复到 inbox-a/
  |                                       |
```

**设置**：

在每个 session 启动时创建 durable cron：

```
创建 durable recurring cron（每 1 分钟）：
检查 .agent-coord/inbox-${SESSION_NAME}/ 是否有新的 .jsonl 文件。
如果有，读取消息内容并处理。处理完后将文件移动到 .processed/ 目录。
```

**优点**：
- 零外部依赖——纯 Claude Code 内置功能
- Durable cron 跨 session 存活
- 简单可靠

**限制**：
- 最小 1 分钟延迟（cron 粒度限制）
- 每次轮询都消耗一个 turn 的 token（即使没消息）
- Cron 在 turn 间隙执行——如果用户持续对话，轮询会排队

---

## 方案对比

| | 方案 A (FileChanged+asyncRewake) | 方案 B (Watchdog) | 方案 C (Durable Cron) |
|---|---|---|---|
| 延迟 | ~100ms（OS 文件事件） | 1-2s（session 冷启动） | 1 分钟（cron 粒度） |
| Token 开销 | 低（只在有消息时唤醒） | 中（每次唤醒新 turn） | 高（每次轮询消耗 turn） |
| 外部依赖 | fswatch/inotifywait | 持续运行的 watchdog | 无 |
| 可靠性 | 中（watch 脚本可能死） | 高（watchdog 独立进程） | 高（durable cron 自动恢复） |
| 复杂度 | 中 | 高 | 低 |
| 跨 session | 有限（需要 hook 配置） | 完全（外部路由） | 完全（文件 + cron） |
| 适合场景 | 交互式 session 间协作 | Headless 自动化流水线 | 简单定期同步 |

---

## 推荐

| 场景 | 推荐方案 |
|------|---------|
| 两个交互式 session 协作 | **方案 A**（低延迟，推送唤醒） |
| 多个 headless 实例流水线 | **方案 B**（watchdog 路由，可扩展） |
| 简单状态同步，容忍延迟 | **方案 C**（零依赖，最简单） |
| 不需要实时通讯 | 直接用文件接力（不需要任何方案，最基础的 Tier 1） |

---

## 消息协议约定

无论选哪个方案，消息格式保持一致：

```
目录结构:
.agent-coord/
  |-- inbox-{session-name}/     # 每个 session 的收件箱
  |   |-- msg-{timestamp}.jsonl
  |   +-- .processed/           # 已处理的消息
  |-- outbox/                   # 发件箱（方案 B 用）
  +-- sessions.json             # session 注册表

消息格式 (.jsonl):
{"from": "session-a", "to": "session-b", "timestamp": "2026-04-04T12:00:00Z",
 "type": "result|request|status", "text": "...", "data_path": "results/a.json"}
```

---

## asyncRewake 详解（方案 A 的核心）

这是源码中最被低估的机制。`asyncRewake: true` 的 hook：

1. 在 `SessionStart` 时启动，后台运行（不阻塞 session）
2. 可以运行任意长时间（`timeout: 3600` = 1 小时）
3. 退出码 2 时，**主动注入 task-notification 唤醒模型**
4. hook 的 stdout（如果是 JSON）中的 `additionalContext` 会被注入

```
SessionStart -> asyncRewake hook 启动（后台）
  |
  hook 内部: fswatch/inotifywait 等待文件变化
  |
  文件变化 -> hook 读取消息 -> 输出 JSON 到 stdout -> exit 2
  |
  Claude Code 收到 exit 2 -> 注入 task-notification -> 模型被唤醒
  |
  模型看到 additionalContext 中的消息内容 -> 处理
```

这本质上是一个**从外部向 Claude Code session 推送消息的后门**，不需要 SendMessage 或 Agent Teams。

限制：每次 asyncRewake 触发后 hook 进程退出。如果需要持续监听，需要在 hook 脚本中实现循环，或者在模型处理完后重新设置一个新的 asyncRewake hook。
