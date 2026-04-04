---
id: "004"
title: "OpenClaw 精简版架构 — Conclusion"
concluded: 2026-04-04
plan: ""
---

# OpenClaw 精简版架构 — Conclusion

## Decision Summary

| # | Topic | Decision | Reversibility |
|---|-------|----------|---------------|
| 1 | 核心提取 | Tool 接口 35→8 方法，2 层 compact，保留 hooks + MCP，砍 UI/企业/Anthropic 绑定 | high |
| 2 | 原生通讯 | P2P mesh over Unix Domain Socket，JSON-RPC 2.0，heartbeat + 超时强杀 | medium |
| 3 | Model-agnostic | 内部 adapter 接口 + OpenAI-compatible 标准化，3 个原生 adapter | high |
| 4 | 技术栈 | Go（单二进制、goroutine IPC、3 周 MVP） | medium |
| 5 | MVP 范围 | 10 tools（含 claw_send/claw_request），<5K 行核心 | high |

---

## Topic 1: 核心提取

### 保留（重写）

| 模块 | Claude Code | OpenClaw | 变化 |
|------|-----------|---------|------|
| Tool 接口 | ~35 方法 | 8 方法 | 砍掉所有 render/UI/classifier 方法 |
| buildTool factory | fail-closed defaults | 保留模式 | 同样的 spread + typed defaults |
| Query loop | QueryEngine + query.ts | 单文件 engine.go | 解耦 Claude API |
| Compact | 3 层（micro/session/full） | 2 层（soft/hard） | micro 是 Claude cache 优化，不需要 |
| Permission | 5 级 rule source + ML classifier | 3 级（config/CLI/runtime） | 砍 ML classifier 和企业 MDM |
| Hooks | 7 事件类型 + asyncRewake | 5 事件 + UDS wake | asyncRewake 概念保留为 UDS 事件 |
| MCP | 119K 自定义 client | 官方 SDK | 用 `@modelcontextprotocol/sdk` 或 Go 等价 |
| Tool concurrency | partitionToolCalls() | 保留模式 | read 并行 / write 串行 |

### 砍掉

| 模块 | 行数 | 理由 |
|------|------|------|
| Custom Ink renderer | 251K | 用标准 TUI 库（bubbletea for Go） |
| GrowthBook/feature flags | 散布各处 | Anthropic A/B 测试，不需要 |
| Undercover mode | ~500 | Anthropic 员工专用 |
| Buddy (Tamagotchi) | ~46K | 彩蛋 |
| KAIROS/ULTRAPLAN | gated | 未发布 feature |
| Enterprise MDM | ~2K | 企业管理 |
| autoDream | ~3K | Claude 记忆整合，OpenClaw 不需要 |
| Prompt cache break detection | ~1K | Claude API 特有 |
| Bridge (IDE integration) | 115K | MVP 不需要 |

### 最小 Tool 接口

```go
type Tool struct {
    Name        string
    Description string
    InputSchema json.RawMessage  // JSON Schema
    Execute     func(input map[string]any, ctx *Context) (*Result, error)
    Validate    func(input map[string]any) error           // optional
    Permission  func(input map[string]any, ctx *Context) Permission // optional
    IsReadOnly  func(input map[string]any) bool
    Concurrent  func(input map[string]any) bool
}

type Result struct {
    Output   string
    Error    string
    Metadata map[string]any
}

type Permission int
const (
    Allow Permission = iota
    Ask
    Deny
)
```

---

## Topic 2: 原生通讯架构

### 设计

```
~/.openclaw/
  sockets/
    {claw-name}.sock          # 每个 claw 实例一个 UDS
  registry.json               # 所有活跃 claw 的注册表
```

**传输层**: Unix Domain Socket（macOS/Linux 原生，零依赖，<1ms 延迟）

**协议**: JSON-RPC 2.0 + 4 字节长度前缀

```
[4 bytes: payload length, big-endian][JSON-RPC payload]
```

**发现**: `registry.json` 原子更新（mktemp + mv）

```json
{
  "claws": {
    "researcher": {
      "socket": "/Users/x/.openclaw/sockets/researcher.sock",
      "pid": 12345,
      "started_at": "2026-04-04T12:00:00Z",
      "model": "claude-opus-4-6",
      "status": "active"
    }
  }
}
```

**通讯原语**:

| 原语 | JSON-RPC method | 语义 |
|------|----------------|------|
| send | `claw.send` | 单向通知（fire-and-forget） |
| request | `claw.request` | 请求+响应（default 30s timeout） |
| broadcast | `claw.broadcast` | 发给所有已注册 claw |
| subscribe | `claw.subscribe` | 订阅事件流（tool_call, state_change） |
| peers | `claw.peers` | 列出所有活跃 claw |
| spawn | `claw.spawn` | 请求 orchestrator 创建新 claw |

**生命周期**（修复 Claude Code 的所有缺陷）:

| 问题 | Claude Code | OpenClaw |
|------|------------|---------|
| 状态不可靠 | isActive?: boolean, undefined=active | heartbeat ping 每 30s，60s 无响应=dead |
| Shutdown 依赖模型 | 发 shutdown_request 等模型回应 | 5s drain → 10s SIGTERM → SIGKILL |
| 无超时 | MCP tool 默认 27.8h | 所有 IPC 操作 default 30s timeout |
| 残留清理 | 手动删 config.json | registry sweeper：启动时清理 stale entries（PID 不存在的） |
| 进程崩溃 | isActive 留 true | socket 断开 = 立即从 registry 移除 |

**关键设计**: claw_send 和 claw_request 是**内置 tool**——模型可以直接调用，不需要外部脚本。

```
Model: "我需要让 researcher 分析 auth 模块"
  → Tool call: claw_send(to="researcher", message="分析 src/auth.ts")
  → UDS: connect ~/.openclaw/sockets/researcher.sock
  → JSON-RPC: {"method":"claw.send","params":{"from":"leader","text":"分析 src/auth.ts"}}
  → researcher 模型收到消息，开始工作
```

---

## Topic 3: Model-Agnostic 适配层

### 接口

```go
type Adapter interface {
    Chat(ctx context.Context, req *ChatRequest) (<-chan StreamEvent, error)
    Models() []ModelInfo
    SupportsToolCalling() bool
    SupportsStreaming() bool
}

type ChatRequest struct {
    Model    string
    Messages []Message
    Tools    []Tool
    Options  map[string]any  // temperature, max_tokens, etc.
}

type StreamEvent struct {
    Type     string  // "text_delta", "tool_call", "tool_call_delta", "usage", "done", "error"
    Content  string
    ToolCall *ToolCall
    Usage    *Usage
}
```

### 适配器

| Adapter | Provider | Tool calling | 实现 |
|---------|----------|-------------|------|
| `claude` | Anthropic API | 原生 tool_use | HTTP streaming client |
| `openai` | OpenAI / Azure / LM Studio | 原生 function calling | HTTP streaming client |
| `gemini` | Google Gemini | 原生 function declarations | HTTP streaming client |
| `ollama` | 本地模型 | 部分支持 / XML fallback | HTTP client + XML parser |
| `litellm` | LiteLLM proxy | 透传 | OpenAI-compatible adapter |

**无 tool calling 的降级**:

```
System prompt 注入:
  "你有以下工具可用：
   <tools>
   <tool name="bash">...</tool>
   </tools>
   
   要使用工具，输出：
   <tool_call name="bash">{"command": "ls"}</tool_call>"

Output 解析:
  正则匹配 <tool_call> 标签 → 提取 tool name + args → 执行
```

### Model routing（可选，MVP 后）

```go
type Router struct {
    Rules []RoutingRule
}

type RoutingRule struct {
    Match  func(task string) bool  // 正则或关键词匹配
    Model  string                   // 目标模型
    Reason string
}

// 示例：
// "写测试" → haiku（便宜，模式化）
// "架构设计" → opus（需要深度推理）
// "搜索代码" → haiku（搜索密集型）
```

---

## Topic 4: 技术栈 — Go

### 决策理由

| 维度 | Go | Rust | Python |
|------|-----|------|--------|
| MVP 时间 | ~3 周 | ~7 周 | ~2 周 |
| UDS IPC | stdlib `net` | tokio + Arc\<RwLock\> | asyncio（sync footgun） |
| LLM SDK | 无官方，手写 HTTP | 无官方 | 全部官方 |
| 分发 | 单静态二进制 | 单静态二进制 | pipx（需要 Python） |
| 插件 | subprocess / MCP | subprocess / MCP | 动态 import |
| 社区贡献 | 低门槛 | 高门槛 | 低门槛 |

**Go 胜出理由**: 分发是 CLI 产品的核心属性。`brew install openclaw` 或 `curl | sh` 是最低摩擦的用户体验。Python 做不到这一点。Rust 的 MVP 时间是 Go 的 2 倍+。Go 的 goroutine + channel 天然适合并行 tool 执行 + N 个 UDS 连接 + LLM streaming。

**Go 的 LLM SDK 缺口**: 无官方 Anthropic/Gemini Go SDK。需要手写 HTTP streaming client——每个 adapter 约 300 行。这是一次性成本，API 稳定后很少改动。

### 依赖

```
go 1.22+
github.com/spf13/cobra          # CLI framework
github.com/charmbracelet/bubbletea  # TUI (optional)
github.com/charmbracelet/lipgloss   # styling
golang.org/x/term                # terminal detection
```

零 CGO，纯 Go，交叉编译无痛。

### 项目结构

```
openclaw/
  cmd/
    claw/main.go            # CLI entry point
  internal/
    engine/                  # query loop
      engine.go              # core turn loop
      compact.go             # context compaction
    adapter/                 # LLM adapters
      adapter.go             # interface
      claude.go
      openai.go
      ollama.go
    tool/                    # tool system
      tool.go                # Tool type + buildTool
      registry.go            # tool pool assembly
      bash.go
      file_read.go
      file_write.go
      file_edit.go
      glob.go
      grep.go
      web_fetch.go
      mcp.go                 # MCP bridge tool
      claw_send.go           # IPC: send message
      claw_request.go        # IPC: request/response
    ipc/                     # inter-claw communication
      server.go              # UDS listener
      client.go              # UDS dialer
      registry.go            # peer discovery
      protocol.go            # JSON-RPC framing
    permission/              # permission system
      permission.go
      rules.go
    hook/                    # hook system
      hook.go
      runner.go
    config/                  # settings
      config.go
  pkg/
    mcp/                     # MCP client (or use SDK)
  configs/
    default.toml             # default settings
```

目标：core < 5K 行。

---

## MVP v0.1 Scope

### 包含

- 10 tools: bash, file_read, file_write, file_edit, glob, grep, web_fetch, mcp_call, **claw_send, claw_request**
- 3 adapters: Claude, OpenAI, Ollama
- UDS IPC with registry, heartbeat, timeout
- 2-tier compact (soft warning + hard summarize)
- 3-tier permissions (config/CLI/runtime)
- 5 hook events (PreToolUse, PostToolUse, SessionStart, SessionEnd, FileChanged)
- Print mode (`claw -p`) + session resume (`claw --resume`)
- TOML config file

### 不包含

- Web UI
- Windows 支持
- IDE integration
- Plugin marketplace
- Cloud sync
- Enterprise auth
- Model routing（MVP 后）

### 差异化

| | aider | cline | OpenHands | Claude Code | **OpenClaw** |
|---|---|---|---|---|---|
| CLI native | Yes | No (IDE) | No (Web) | Yes | **Yes** |
| Multi-agent | No | In-process | Cloud tasks | In-process (fragile) | **Native IPC (UDS)** |
| Model-agnostic | Partial | Yes | Yes | No | **Yes** |
| Peer identity | N/A | N/A | No (tasks) | Name@team (memory) | **Name (socket)** |
| Open source | Yes | Yes | Yes | No | **Yes** |
| Process isolation | N/A | No | Yes (VM) | No (AsyncLocal) | **Yes (OS process)** |
| Crash resilience | N/A | N/A | Yes | No (all die) | **Yes (independent)** |

**核心卖点**: "Multiple AI agents that can actually talk to each other, across independent processes, with any model, from a single CLI."

---

## Topic 6: 跨设备通讯与分布式

### 传输层抽象

协议层（JSON-RPC 2.0）不变，只换传输层：

```go
type Transport interface {
    Listen(addr string) (net.Listener, error)
    Dial(addr string) (net.Conn, error)
}

type UDSTransport struct{}    // 本地（v0.1）
type TCPTransport struct{}    // 跨设备直连（v0.2）
type NATSTransport struct{}   // broker-based（v0.3+）
```

上层 claw_send / claw_request / claw_peers 完全不变。模型不需要知道对方是本地还是远程。

### 三层方案

**Tier 1：TCP 直连**

```toml
# claw.toml
[ipc]
transport = "tcp"
listen = "0.0.0.0:7700"
tls_cert = "~/.openclaw/certs/claw.pem"
tls_key = "~/.openclaw/certs/claw-key.pem"

[peers]
tester = "192.168.1.10:7700"
researcher = "192.168.1.11:7700"
```

- 发现：静态配置
- 安全：mTLS
- 适合：2-3 台固定机器，同一 LAN

**Tier 2：Tailscale mesh（推荐）**

```
Machine A                    Machine B                    Machine C
  claw 100.x.x.1:7700         claw 100.x.x.2:7700         claw 100.x.x.3:7700
      |                            |                            |
      +────── Tailscale WireGuard mesh（自动加密，自动穿 NAT）───+
```

```toml
[ipc]
transport = "tcp"
listen = "100.64.0.1:7700"  # tailscale IP

[discovery]
method = "dns"  # tailscale MagicDNS: researcher.tailnet → 100.64.0.2
```

- 零运维，自动穿 NAT，自动加密
- 适合：个人多设备、小团队

**Tier 3：消息 broker**

```toml
[ipc]
transport = "nats"
broker = "nats://nats.example.com:4222"

[discovery]
method = "broker"  # NATS subject routing 自动发现
```

- NATS：轻量（单二进制 15MB），pub/sub + request/reply + JetStream 持久化
- Redis Cluster：如果已有 Redis 基建
- 适合：大规模、云端部署

### 发现机制对比

| 方案 | 适合 | 配置 |
|------|------|------|
| 静态 config | 2-3 台固定机器 | peer 地址写 claw.toml |
| mDNS/Bonjour | 同一 LAN 自动发现 | Go multicast，macOS 原生 |
| Tailscale MagicDNS | 跨 NAT 多设备 | 零配置（装 tailscale 就行） |
| NATS subject routing | 大规模动态 | broker 地址 + subject 命名空间 |
| Consul/etcd | 企业级 | 服务注册 + 健康检查 |

### 安全

| 传输 | 加密 | 认证 |
|------|------|------|
| UDS | 不需要（本地进程间） | 文件系统权限 |
| TCP 直连 | mTLS | 证书双向验证 |
| Tailscale | WireGuard（自动） | Tailscale 身份（自动） |
| NATS | TLS | token / nkey / JWT |

### 分布式场景示例

**场景：Mac（本地开发）+ Linux 服务器（GPU 推理）**

```
MacBook (Tailscale 100.64.0.1)           GPU Server (Tailscale 100.64.0.2)
  claw --name leader                       claw --name gpu-worker
  model: claude-opus                       model: ollama/codestral
  |                                        |
  claw_send(to="gpu-worker",              收到任务 → 用本地 GPU 推理
    message="分析这段代码的性能")          → claw_send(to="leader", result)
```

**场景：团队协作（3 人各自机器上跑 claw）**

```
开发者 A (researcher)     开发者 B (reviewer)     CI Server (tester)
  claw --name researcher    claw --name reviewer    claw --name tester
  model: opus               model: sonnet           model: haiku
      |                         |                       |
      +──────────── NATS broker (nats.team.internal) ───+

A: claw_send(to="reviewer", message="PR #42 准备好了，请 review")
B: claw_send(to="tester", message="review 通过，请跑测试")
C: claw_send(to="researcher", message="测试通过，3/3 passed")
```

### 版本路线

| 版本 | 传输 | 发现 | 复杂度 |
|------|------|------|--------|
| v0.1 | UDS only | registry.json | 低 |
| v0.2 | + TCP 直连 | + 静态 config | 低（加个 TCPTransport） |
| v0.2.1 | + Tailscale | + MagicDNS | 零（TCP 上跑，Tailscale 透明） |
| v0.3 | + NATS | + broker discovery | 中 |

Transport interface 是第一天就设计好的，后续加传输层不改上层任何代码。

---

## Claw <-> Claude Code 桥接

详见 [Discussion 005](../005-claw-cc-bridge/conclusion.md)。

核心设计：
- CC → Claw：MCP tool call → Claw UDS/TCP socket（同步，<5ms）
- Claw → CC：写文件 → asyncRewake hook exit 2 → 唤醒 CC 模型（异步，~1s）
- `claw mcp-bridge <name>` 命令做 stdio↔UDS 协议转换

---

## Team Composition

| Agent | Role | Backend |
|-------|------|---------|
| TL | Moderator | Claude |
| architect | Architecture design | Claude |
| code-researcher | Source code analysis + competitive survey | Claude |
| codex-proxy | Tech stack + positioning | Codex (o4-mini) |

## Process Metadata
- Discussion rounds: 2
- Topics: 4 (+ tech stack deep-dive sub-round)
- Autonomous decisions: 4
- User escalations: 1 (tech stack choice — user excluded TS, confirmed Go/Rust/Python)

## Next Steps
- `/ae:plan` 规划 OpenClaw v0.1 的具体实现步骤
- 可以先开 repo，搭 Go module skeleton
- 优先实现: engine + 3 tools (bash/read/edit) + 1 adapter (OpenAI) + UDS IPC
