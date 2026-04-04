---
id: "005"
title: "Claw <-> Claude Code 通讯桥接 — Conclusion"
concluded: 2026-04-04
plan: ""
---

# Claw <-> Claude Code 通讯桥接 — Conclusion

## 核心设计：双通道，无 daemon

```
┌─ Claude Code Session ──────────────┐         ┌─ OpenClaw Instance ──────────────┐
│                                     │         │                                   │
│  CC → Claw:                         │         │  Claw UDS Socket                  │
│  MCP client ──────────────────────────────────→  ~/.openclaw/sockets/claw-a.sock  │
│  调用 claw_send / claw_request      │  UDS    │  双人格：JSON-RPC + MCP server    │
│  (同步，<5ms)                       │         │                                   │
│                                     │         │                                   │
│  Claw → CC:                         │         │  Claw → CC:                       │
│  asyncRewake hook ←─────────────────────────── 写文件到 ~/.claude/inbox/{sid}/   │
│  检测文件 → exit 2 → 唤醒模型        │  file   │  (原子写：mktemp + mv)            │
│  additionalContext 注入消息          │         │                                   │
│                                     │         │                                   │
└─────────────────────────────────────┘         └───────────────────────────────────┘
```

**不需要 bridge daemon。** 两个方向各用最优的原生通道，无中间进程。

---

## CC → Claw：MCP Server on UDS

**Claw 把自己暴露为 MCP server**，Claude Code 用原生 MCP client 直接调用。

### Claw 侧

Claw 的 UDS socket 同时服务两种协议：
- 来自其他 Claw 的 JSON-RPC（claw-to-claw）
- 来自 Claude Code 的 MCP tool calls（CC-to-claw）

通过请求头区分：MCP 用标准 MCP framing，JSON-RPC 用 4 字节长度前缀。

暴露的 MCP tools：

| MCP Tool | 作用 | 返回 |
|----------|------|------|
| `claw_send` | 发送消息给此 claw | `{status: "delivered"}` |
| `claw_request` | 发送并等待回复（default 30s timeout） | `{response: "..."}` |
| `claw_status` | 查询此 claw 的状态 | `{status: "active", model: "...", task: "..."}` |
| `claw_peers` | 列出所有已注册的 claw 实例 | `[{name, socket, status}]` |
| `poll_inbox` | CC 主动拉取待处理消息 | `[{from, text, timestamp}]` |

### CC 侧配置

```json
// .mcp.json（项目级）
{
  "mcpServers": {
    "claw-researcher": {
      "command": "socat",
      "args": ["STDIO", "UNIX-CONNECT:/Users/ckai/.openclaw/sockets/researcher.sock"],
      "env": {}
    }
  }
}
```

或者 Claw 提供一个 MCP adapter 命令：

```json
{
  "mcpServers": {
    "claw-researcher": {
      "command": "claw",
      "args": ["mcp-bridge", "researcher"],
      "env": {}
    }
  }
}
```

`claw mcp-bridge <name>` 启动一个 stdio MCP server，内部连接到目标 claw 的 UDS socket，做 MCP↔JSON-RPC 协议转换。

### CC 模型调用示例

```
Model: "让 researcher 分析 auth 模块"
  → Tool call: mcp__claw-researcher__claw_send(message="分析 src/auth.ts 的安全问题")
  → MCP client → socat/mcp-bridge → UDS → Claw researcher 收到消息
  → 返回: {status: "delivered"}

Model: "researcher 分析完了吗？"
  → Tool call: mcp__claw-researcher__claw_status()
  → 返回: {status: "active", task: "分析 src/auth.ts"}

Model: "等 researcher 的结果"
  → Tool call: mcp__claw-researcher__claw_request(message="返回你的分析结果", timeout=60)
  → 阻塞等待（最多 60s）→ researcher 返回结果
  → 返回: {response: "发现 3 个安全问题: 1. ..."}
```

**延迟**: <5ms（UDS 本地通讯）
**可靠性**: Claw 不在线 → 立即 connection refused，CC 看到 MCP 错误

---

## Claw → CC：文件 + asyncRewake

**Claw 写文件到 CC 的 inbox 目录，asyncRewake hook 检测到后唤醒 CC 模型。**

### Claw 侧

```go
// Claw 发消息给 Claude Code session
func SendToCC(sessionID, message string) error {
    inbox := filepath.Join(os.Getenv("HOME"), ".claude", "inbox", sessionID)
    os.MkdirAll(inbox, 0755)
    
    msg := Message{
        From:      clawName,
        Text:      message,
        Timestamp: time.Now().UTC().Format(time.RFC3339),
    }
    
    // 原子写
    tmp, _ := os.CreateTemp(inbox, ".tmp.*")
    json.NewEncoder(tmp).Encode(msg)
    tmp.Close()
    os.Rename(tmp.Name(), filepath.Join(inbox, fmt.Sprintf("msg-%d.jsonl", time.Now().UnixNano())))
    
    return nil
}
```

### CC 侧 hook

```json
// ~/.claude/settings.json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "bash /Users/ckai/.claude/scripts/cc-inbox-watch.sh",
        "asyncRewake": true,
        "timeout": 3600
      }]
    }]
  }
}
```

**cc-inbox-watch.sh**:

```bash
#!/bin/bash
# 监听 CC inbox，有新消息时 exit 2 唤醒模型
INBOX="$HOME/.claude/inbox/${CLAUDE_SESSION_ID:-default}"
mkdir -p "$INBOX"

# macOS: stat 轮询（1s 间隔），或 fswatch 如果可用
ELAPSED=0
while [ "$ELAPSED" -lt 3600 ]; do
  UNREAD=$(find "$INBOX" -maxdepth 1 -name "*.jsonl" 2>/dev/null | head -5)
  if [ -n "$UNREAD" ]; then
    MESSAGES=""
    for f in $UNREAD; do
      MSG=$(cat "$f" 2>/dev/null || true)
      [ -n "$MSG" ] && MESSAGES="${MESSAGES}${MSG}\n"
      mv "$f" "$INBOX/.processed/" 2>/dev/null || true
    done
    mkdir -p "$INBOX/.processed"
    printf '{"hookSpecificOutput":{"additionalContext":"[Claw Message]\\n%s"}}\n' \
      "$(printf '%b' "$MESSAGES" | head -c 2000)"
    exit 2  # 唤醒模型
  fi
  sleep 1
  ELAPSED=$((ELAPSED + 1))
done
exit 0
```

### CC 模型看到的

```
<system-reminder>
[Claw Message]
{"from":"researcher","text":"auth 模块分析完成，发现 3 个问题：1. token 无过期检查 2. ...","timestamp":"2026-04-04T21:30:00Z"}
</system-reminder>
```

**延迟**: ~1s（stat 轮询）或 ~100ms（fswatch）
**可靠性**: 消息在文件系统中持久化，CC 离线时消息排队，下次唤醒时批量处理

### 备选：poll_inbox（无需 asyncRewake）

如果 asyncRewake 不可用或不想配 hook，CC 可以主动拉取：

```
Model: "检查一下有没有 claw 发来的消息"
  → Tool call: mcp__claw-researcher__poll_inbox()
  → 返回: [{from: "researcher", text: "分析完成...", timestamp: "..."}]
```

或者用 durable cron：
```
/loop 1m 调用 mcp__claw-researcher__poll_inbox()，如果有消息就处理
```

---

## 完整工作流示例

### 场景：CC 做主控，Claw 做并行 worker

```
Terminal 1: Claude Code (leader)
Terminal 2: claw --name researcher --model claude-opus
Terminal 3: claw --name tester --model haiku

CC 配置 .mcp.json:
{
  "mcpServers": {
    "claw-researcher": {"command": "claw", "args": ["mcp-bridge", "researcher"]},
    "claw-tester": {"command": "claw", "args": ["mcp-bridge", "tester"]}
  }
}
```

**CC 分配任务**:
```
> 让 researcher 分析 auth 模块，让 tester 写 auth 测试

Model:
  mcp__claw-researcher__claw_send(message="分析 src/auth/ 模块，找出安全问题")
  mcp__claw-tester__claw_send(message="为 src/auth/ 写单元测试")
```

**Claw 完成后通知 CC**:
```
// researcher 完成 → 写文件到 CC inbox
SendToCC(ccSessionID, "auth 分析完成，发现 3 个问题，详见 .agent-coord/results/auth-analysis.json")

// CC 的 asyncRewake hook 检测到 → 唤醒模型
// CC 模型看到: [Claw Message] auth 分析完成...
```

**CC 综合**:
```
> 读取 researcher 和 tester 的结果，综合报告

Model: 读取 .agent-coord/results/auth-analysis.json 和 auth-tests.json
       综合报告...
```

### 场景：Claw 做主控，CC 做 Claude 专用 worker

```
Terminal 1: claw --name leader --model gpt-4o
Terminal 2: Claude Code (作为 Claude 专家)

Claw leader 通过文件+asyncRewake 给 CC 发任务：
  SendToCC(ccSessionID, "用你的 Claude 能力分析这段代码的 extended thinking")

CC 完成后通过 MCP 回报：
  CC model: mcp__claw-leader__claw_send(message="分析完成: ...")
```

---

## 消息协议

两个方向使用统一的消息格式：

```json
{
  "from": "researcher",
  "to": "cc-session-abc123",
  "msg_id": "msg-1712345678-12345",
  "reply_to": null,
  "timestamp": "2026-04-04T21:30:00Z",
  "type": "result",
  "text": "分析完成",
  "data_path": ".agent-coord/results/auth.json",
  "metadata": {}
}
```

---

## 对比

| | CC → Claw (MCP) | Claw → CC (file+hook) |
|---|---|---|
| 延迟 | <5ms | ~1s (poll) / ~100ms (fswatch) |
| 模式 | 同步 request/response | 异步 push |
| 可靠性 | 实时（Claw 不在=立即错误） | 持久（消息排队，不丢） |
| 配置 | .mcp.json 注册 | asyncRewake hook 或 cron |
| 模型感知 | 原生 MCP tool call | additionalContext 注入 |

**不对称是 feature，不是 bug。** CC→Claw 是同步查询/命令（"去做这个"），Claw→CC 是异步通知（"我做完了"）。这正好匹配实际工作流。

---

## 实现优先级

| 优先级 | 组件 | 工作量 |
|--------|------|-------|
| P0 | Claw MCP server (UDS) | Claw 核心功能，已在 IPC 设计中 |
| P0 | `claw mcp-bridge` 命令 | ~100 行 Go，stdio↔UDS 转换 |
| P1 | CC inbox watch 脚本 | ~30 行 bash（已有模板） |
| P1 | Claw SendToCC 函数 | ~20 行 Go |
| P2 | poll_inbox MCP tool | asyncRewake 的 pull 备选 |
| P2 | CC→Claw 的 socat fallback | 无需 MCP 配置的快速测试方案 |

---

## Next Steps
- 这个桥接设计是 OpenClaw 的一部分，不需要独立实施
- 在 `/ae:plan` OpenClaw v0.1 时把 `claw mcp-bridge` 命令和 CC inbox watch 脚本包含在 MVP scope 中
