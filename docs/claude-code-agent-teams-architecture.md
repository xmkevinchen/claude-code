# Claude Code Agent Teams 实现架构

基于源码的完整分析：Team 生命周期、通讯机制、任务协调、隔离模型。

---

## 架构总览

```
┌─ Team Lead（主 session 进程）────────────────────────────────────────┐
│                                                                      │
│  AppState                                                            │
│  ├── teamContext: { teamName, teammates Map, leadAgentId }           │
│  ├── tasks: Map<agentId, TaskState>  ← 所有 in-process agent 共享    │
│  ├── agentNameRegistry: Map<name, agentId>  ← SendMessage 路由       │
│  └── inbox: [] ← 接收 teammate 消息                                  │
│                                                                      │
│  ┌─ In-Process Teammate A ─────────────┐                             │
│  │  AsyncLocalStorage 隔离              │                             │
│  │  独立 AbortController               │                             │
│  │  独立对话历史                         │                             │
│  │  共享: API client, MCP, tools        │                             │
│  └──────────────────────────────────────┘                             │
│                                                                      │
│  ┌─ In-Process Teammate B ─────────────┐                             │
│  │  AsyncLocalStorage 隔离              │                             │
│  │  ...                                │                             │
│  └──────────────────────────────────────┘                             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
         ↕ 文件系统通讯                    ↕ 内存通讯（in-process）
┌─ 磁盘状态 ───────────────────────────────────────────────────────────┐
│  ~/.claude/teams/{name}/config.json        ← Team 配置 + 成员列表    │
│  ~/.claude/teams/{name}/inboxes/{agent}.json  ← 邮箱（JSON 数组）    │
│  ~/.claude/tasks/{name}/{id}.json          ← 共享任务列表             │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 一、Team 生命周期

### 创建（TeamCreate）

**文件**: `src/tools/TeamCreateTool/TeamCreateTool.ts:128-236`

```
TeamCreate({ team_name: "my-team", description: "..." })
  ↓
1. 检查 appState.teamContext — 一个 leader 只能领导一个 team
2. generateUniqueTeamName() — 名称冲突时生成 word-slug 替代
3. leadAgentId = "team-lead@my-team"
4. 写入 TeamFile → ~/.claude/teams/my-team/config.json
5. registerTeamForSessionCleanup() — 注册退出清理
6. resetTaskList() + ensureTasksDir() — 创建任务目录
7. setLeaderTeamName() — 让 getTaskListId() 解析到 team
8. 更新 AppState.teamContext
```

**Team 配置文件格式**（`src/utils/swarm/teamHelpers.ts:64-90`）：

```json
{
  "name": "my-team",
  "description": "...",
  "createdAt": "2026-04-04T12:00:00Z",
  "leadAgentId": "team-lead@my-team",
  "leadSessionId": "uuid-xxx",
  "members": [
    {
      "agentId": "researcher@my-team",
      "name": "researcher",
      "agentType": "ae:research:archaeologist",
      "model": "opus",
      "backendType": "in-process",
      "isActive": true,
      "mode": "default",
      "cwd": "/path/to/project",
      "worktreePath": null,
      "sessionId": "uuid-yyy",
      "tmuxPaneId": null,
      "subscriptions": []
    }
  ],
  "hiddenPaneIds": [],
  "teamAllowedPaths": []
}
```

### 销毁（TeamDelete）

**文件**: `src/tools/TeamDeleteTool/TeamDeleteTool.ts:71-135`

```
TeamDelete({ team_name: "my-team" })
  ↓
1. 检查所有成员 isActive — 有活跃成员则拒绝（必须先 shutdown）
2. cleanupTeamDirectories() — 删除 worktrees、team 目录、task 目录
3. unregisterTeamForSessionCleanup()
4. 清空 AppState.teamContext + inbox
```

### Leader 退出时的清理

**文件**: `src/utils/swarm/teamHelpers.ts:576-590`

`cleanupSessionTeams()` 注册在 `gracefulShutdown` 中：
- 正常退出：发送 shutdown_request → 等待 response → 清理
- 异常退出（SIGINT/SIGTERM）：`killOrphanedTeammatePanes()` 强杀 tmux pane 进程 + `cleanupTeamDirectories()`

---

## 二、Agent 生成与后端

### Agent ID 格式

**文件**: `src/utils/agentId.ts`

```
Agent ID:    "agentName@teamName"     ← 确定性、人类可读
Request ID:  "shutdown-1702500000@researcher@my-team"
Team Lead:   "team-lead@my-team"      ← 固定格式
```

### 后端选择优先级

**文件**: `src/utils/swarm/backends/registry.ts:136-253`

```
1. 在 tmux 内          → TmuxBackend（独立 tmux pane）
2. 在 iTerm2 + it2 CLI → ITermBackend（原生 iTerm2 pane）
3. 在 iTerm2 无 it2    → tmux fallback
4. 非 tmux/iTerm2，    → tmux external session
   但有 tmux 可用
5. 非交互模式（-p）     → 强制 in-process
6. CLAUDE_CODE_TEAMMATE_MODE=in-process → 强制 in-process
```

### In-Process 生成流程

**文件**: `src/utils/swarm/spawnInProcess.ts:104-215`

```
spawnTeammate()
  ↓
1. formatAgentId(name, teamName) → "researcher@my-team"
2. 创建独立 AbortController（不与 leader 绑定）
3. createTeammateContext() → 存入 AsyncLocalStorage
4. 创建 InProcessTeammateTaskState → 注册到 AppState.tasks
5. startInProcessTeammate()（fire-and-forget 异步）
   ↓
   runAgent() → 独立对话循环
   完成一个 turn → sendIdleNotification() → 进入 idle 轮询
```

### 三种后端对比

| | In-Process | Tmux Pane | iTerm2 Pane |
|---|---|---|---|
| 进程 | 同一 Node.js 进程 | 独立 Claude 进程 | 独立 Claude 进程 |
| 隔离 | AsyncLocalStorage | 完全进程隔离 | 完全进程隔离 |
| 通讯 | 内存 + 文件 mailbox | 纯文件 mailbox | 纯文件 mailbox |
| 共享 | API client, MCP, tools | 无 | 无 |
| 开销 | 低（同进程） | 中（新进程） | 中（新进程） |
| 可视化 | 无独立面板 | tmux pane 可见 | iTerm2 tab 可见 |

---

## 三、通讯机制（SendMessage）

### 消息路由

**文件**: `src/tools/SendMessageTool/SendMessageTool.ts:741-912`

```
SendMessage({ to: "researcher", message: "开始任务" })
  ↓
路由优先级:
  1. bridge:<session-id>  → postInterClaudeMessage()（跨机器，经 Anthropic 服务器）
  2. uds:<socket-path>    → sendToUdsSocket()（跨 session，Unix Domain Socket）
  3. 名字匹配 in-process agent（运行中）→ queuePendingMessage()（内存注入）
  4. 名字匹配 in-process agent（已停止）→ resumeAgentBackground()（自动恢复）
  5. to: "*"              → handleBroadcast()（写入所有成员 mailbox）
  6. 名字匹配 teammate     → writeToMailbox()（文件写入）
```

### Mailbox 实现

**文件**: `src/utils/teammateMailbox.ts`

**存储位置**: `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

```json
[
  {
    "from": "team-lead@my-team",
    "text": "开始处理 PR #42",
    "timestamp": "2026-04-04T12:00:00Z",
    "read": false,
    "color": "blue",
    "summary": "assign PR task"
  }
]
```

**并发控制**: `proper-lockfile`，10 次重试，5-100ms 退避。写操作：加锁 → 读现有 → 追加 → 写回 → 释放锁。

### In-Process 消息投递

```
发送方调用 SendMessage
  ↓
目标 agent 在运行中？
  ├─ 是 → queuePendingMessage()
  │       添加到 AppState.tasks[agentId].pendingUserMessages[]
  │       agent 在下一次工具调用间隙收到
  │
  └─ 否（idle）→ writeToMailbox()
                  写入 ~/.claude/teams/{name}/inboxes/{agent}.json
                  agent 在 500ms 轮询中读到
```

### 消息在 agent 端的呈现

消息注入为 XML 格式的 user-role message：

```xml
<teammate-message teammate_id="team-lead" color="blue" summary="assign task">
  开始处理 PR #42 的 review comments
</teammate-message>
```

### 结构化消息

**文件**: `SendMessageTool.ts:46-64`（Zod discriminated union）

| 类型 | 方向 | 用途 |
|------|------|------|
| `shutdown_request` | Leader → Teammate | 请求关闭 |
| `shutdown_response` | Teammate → Leader | 确认/拒绝关闭（确认后触发 `AbortController.abort()`） |
| `plan_approval_response` | Leader → Teammate | 批准/拒绝 teammate 的计划 |

纯文本消息用于普通沟通。结构化消息**不能 broadcast**（`to: "*"` 只接受 string）。

### 广播

**文件**: `SendMessageTool.ts:191-265`

```
SendMessage({ to: "*", message: "全员注意", summary: "broadcast" })
  ↓
读取 teamFile.members
  ↓
排除发送者自己
  ↓
对每个成员调用 writeToMailbox()
```

---

## 四、Idle/Wake 机制

### Idle 流程

**In-Process**（`src/utils/swarm/inProcessRunner.ts`）：

```
runAgent() 完成一个 turn
  ↓
sendIdleNotification()
  ↓
写入 leader mailbox:
  { type: "idle_notification", from: "researcher",
    idleReason: "available", completedTaskId: "3",
    completedStatus: "completed" }
  ↓
进入 waitForNextPromptOrShutdown()
  ↓
500ms 轮询循环:
  1. 检查 AppState.tasks[id].pendingUserMessages（内存，最快）
  2. 检查 mailbox 文件（文件 I/O）
  3. 尝试 tryClaimNextTask()（自动认领可用任务）
  4. 收到 shutdown_request → 响应 + abort
  5. 收到普通消息 → 唤醒，开始新 turn
```

**Pane-Based**（`src/utils/swarm/teammateInit.ts:98-128`）：

```
Claude 进程的 session Stop 事件
  ↓
addFunctionHook('Stop', callback)
  ↓
callback:
  1. setMemberActive(teamName, name, false) → config.json 中 isActive: false
  2. createIdleNotification() → 写入 leader mailbox
```

### Wake 触发条件

| 触发源 | 机制 | 延迟 |
|--------|------|------|
| Leader SendMessage | 内存注入（in-process）或 mailbox 写入 | <1ms / <500ms |
| Peer SendMessage | mailbox 写入 | <500ms |
| 新 Task 变为可用 | `tryClaimNextTask()` 轮询 | <500ms |
| Plan approval | 结构化消息通过 mailbox | <500ms |

### Idle 通知格式

```json
{
  "type": "idle_notification",
  "from": "researcher",
  "timestamp": "2026-04-04T12:05:00Z",
  "idleReason": "available",
  "completedTaskId": "3",
  "completedStatus": "completed",
  "peerDmSummary": "sent findings to standards-expert"
}
```

`idleReason` 值：`available`（正常完成）、`interrupted`（被中断）、`failed`（出错）。

---

## 五、任务协调系统

### 任务存储

**文件**: `src/utils/tasks.ts:221-228`

```
~/.claude/tasks/{sanitized-team-name}/
  ├── 1.json
  ├── 2.json
  ├── 3.json
  └── .highwatermark    ← 防止 resetTaskList() 后 ID 复用
```

### 任务格式

```json
{
  "id": "3",
  "subject": "处理 PR #42 review comments",
  "description": "读取 comments，修改代码，commit 并 push",
  "status": "in_progress",
  "owner": "researcher@my-team",
  "blocks": [],
  "blockedBy": ["1", "2"],
  "activeForm": null,
  "metadata": {}
}
```

### 任务状态机

```
pending ──→ in_progress ──→ completed
  │              │
  │              └──→ pending（释放 ownership）
  │
  └──→ completed（直接完成，如果不需要执行）
```

### 任务认领流程

**文件**: `src/utils/swarm/inProcessRunner.ts:624-656`

```
tryClaimNextTask()（在 idle 轮询中调用）
  ↓
1. listTasks() — 读取所有任务文件
2. findAvailableTask() — 找 status=pending, 无 owner, blockedBy 全部 completed
3. claimTask() — 文件锁保护，设置 owner = 自己的 agentName
4. updateTask() — status → in_progress
5. 返回任务 → agent 用任务内容作为下一个 prompt
```

### 并发控制

文件锁：30 次重试，5-100ms 退避——为 10+ 并发 swarm agent 设计。

### Task List ID 解析优先级

```
1. CLAUDE_CODE_TASK_LIST_ID env var
2. AsyncLocalStorage 中的 teammateCtx.teamName（in-process teammate）
3. CLAUDE_CODE_TEAM_NAME env var（pane-based teammate）
4. leaderTeamName 模块变量（TeamCreate 设置）
5. Session ID（独立 session fallback）
```

---

## 六、隔离模型

### In-Process Teammate 隔离

| 组件 | 隔离级别 | 机制 |
|------|---------|------|
| 对话历史 | 完全隔离 | 每个 agent 独立 `promptMessages` |
| AbortController | 完全隔离 | leader 中断不影响 teammate |
| CWD | 可隔离 | `cwd` 参数或 `isolation: "worktree"` |
| API client | 共享 | 同进程，复用连接 |
| MCP connections | 共享 | 同进程，复用连接 |
| Tool 列表 | 共享（可裁剪） | `allowedTools` 参数 |
| 权限 | 可独立配置 | `mode` 参数（default/plan/auto/bypassPermissions） |
| 文件系统 | 共享（或 worktree 隔离） | `isolation: "worktree"` |
| 上下文压缩 | 完全隔离 | 每个 agent 独立触发 autocompact |
| AsyncLocalStorage | 完全隔离 | `TeammateContext` per agent |

### 权限继承

**默认**: Teammate 获得 `permissionMode: 'default'`（无论 leader 是什么模式）。

**覆盖方式**:
- Agent 工具的 `mode` 参数：`acceptEdits`, `auto`, `bypassPermissions`, `default`, `dontAsk`, `plan`
- Leader 可在 UI 中用 Shift+Tab 切换 teammate 的权限模式
- `planModeRequired` flag 强制 teammate 进入 plan 模式

### Teammate System Prompt

**文件**: `src/utils/swarm/teammatePromptAddendum.ts`

Teammate 获得完整的主 system prompt + 额外附录：

```
"You are running as an agent in a team.
 To communicate... you MUST use the SendMessage tool."
```

团队必要工具无条件注入（即使 `allowedTools` 有限制）：
- `SendMessage`
- `TeamCreate` / `TeamDelete`
- `TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate`

---

## 七、Coordinator Mode vs Agent Teams

| | Coordinator Mode | Agent Teams |
|---|---|---|
| 激活方式 | `CLAUDE_CODE_COORDINATOR_MODE=1` | `TeamCreate` 工具 |
| 构建时 gate | `feature('COORDINATOR_MODE')` | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` |
| Worker 身份 | UUID（临时） | `name@team`（确定性） |
| Worker 生命 | 一次性（完成即销毁） | 持久（idle → wake 循环） |
| 通讯方式 | `<task-notification>` XML | 文件 mailbox + SendMessage |
| 团队文件 | 无 | `~/.claude/teams/{name}/config.json` |
| 共享任务列表 | 无 | `~/.claude/tasks/{name}/` |
| Worker 工具 | 受限（不给 TeamCreate/SendMessage） | 完整（含团队工具） |
| Worker 系统 prompt | 无额外 addendum | Teammate addendum |
| 适合场景 | 一次性分发任务 | 长时间协作、迭代式工作 |

**可以同时使用**：Coordinator mode 通过 env var 启用，Agent Teams 通过 `AppState.teamContext` 管理。两者正交。但 coordinator worker 的工具白名单中没有 SendMessage 和 Team 工具。

---

## 八、Team Memory Sync

**文件**: `src/services/teamMemorySync/index.ts`

这**不是** Agent Teams 内部的实时协调——而是组织级别的 CLAUDE.md 共享记忆同步。

```
本地 ~/.claude/memory/ ←→ Anthropic API /api/claude_code/team_memory?repo={owner/repo}
```

- 按 repo 隔离（git remote hash）
- 所有 org 成员共享
- 拉取：服务端优先
- 推送：仅 delta（按内容 hash）
- 文件删除不同步到服务端
- 推送前有 secret scanning
- 限制：单文件 250KB，PUT body 200KB

### Agent Teams 内部的状态共享

实时协调使用以下机制（不经过 Anthropic 服务器）：

| 共享通道 | 存储 | 访问方式 |
|---------|------|---------|
| Team config | `~/.claude/teams/{name}/config.json` | 文件锁读写 |
| Mailbox | `~/.claude/teams/{name}/inboxes/{agent}.json` | 文件锁读写 |
| Task list | `~/.claude/tasks/{name}/{id}.json` | 文件锁读写 |
| In-memory tasks | `AppState.tasks` Map | 直接内存访问（in-process 共享） |
| Agent registry | `AppState.agentNameRegistry` | 直接内存访问 |
| Pending messages | `AppState.tasks[id].pendingUserMessages` | 直接内存访问 |

---

## 九、完整消息流示例

### Leader 发消息给 Teammate

```
Leader: SendMessage({ to: "researcher", message: "分析 auth 模块" })
  ↓
SendMessageTool.call()
  ↓
路由: "researcher" → 查找 appState.agentNameRegistry
  ↓
找到 in-process agent, isActive: true
  ↓
queuePendingMessage() → AppState.tasks["researcher@team"].pendingUserMessages.push(msg)
  ↓
Researcher 在 runAgent() 的工具调用间隙检查到 pendingUserMessages
  ↓
注入为 user message:
  <teammate-message teammate_id="team-lead" color="blue" summary="assign task">
    分析 auth 模块
  </teammate-message>
  ↓
Researcher 的模型收到，开始工作
```

### Teammate 完成任务 → 通知 Leader

```
Researcher: runAgent() 完成当前 turn
  ↓
TaskUpdate({ id: "3", status: "completed" })
  ↓
sendIdleNotification()
  ↓
写入 leader mailbox:
  ~/.claude/teams/my-team/inboxes/team-lead.json
  [{ from: "researcher", type: "idle_notification",
     idleReason: "available", completedTaskId: "3" }]
  ↓
进入 waitForNextPromptOrShutdown()（500ms 轮询）
  ↓
Leader 在下一个 turn 收到:
  <teammate-message teammate_id="researcher" color="green">
    {"type":"idle_notification","from":"researcher",...}
  </teammate-message>
```

### Teammate 之间的 Peer 通讯

```
Researcher: SendMessage({ to: "challenger", message: "这是我的发现..." })
  ↓
writeToMailbox("challenger", msg)
  ↓
~/.claude/teams/my-team/inboxes/challenger.json 追加消息
  ↓
Challenger 在 idle 轮询（500ms）中读到 mailbox 有新消息
  ↓
唤醒 → 注入 teammate-message → 模型处理
  ↓
同时: Researcher 的 idle notification 中包含 peerDmSummary
  → Leader 可以看到 peer 间通讯的摘要
```

### Shutdown 流程

```
Leader: SendMessage({ to: "researcher",
                      message: { type: "shutdown_request", reason: "done" } })
  ↓
写入 researcher mailbox
  ↓
Researcher 在 idle 轮询中收到 shutdown_request
  ↓
Researcher 模型响应:
  SendMessage({ to: "team-lead",
                message: { type: "shutdown_response",
                           request_id: "shutdown-xxx@researcher",
                           approve: true } })
  ↓
approve: true → AbortController.abort() → Researcher 进程终止
  ↓
config.json 中 isActive: false
  ↓
Leader 收到 shutdown_response → 确认 researcher 已关闭
```

---

## 十、性能特征

| 操作 | 延迟 | 瓶颈 |
|------|------|------|
| In-process 消息投递（运行中） | <1ms | 内存操作 |
| Mailbox 写入 | 5-50ms | 文件锁 + I/O |
| Mailbox 轮询 | 500ms 周期 | setInterval |
| Task 认领 | 5-100ms | 文件锁（30 次重试） |
| Teammate 生成（in-process） | 50-200ms | API 首次调用 |
| Teammate 生成（tmux pane） | 1-3s | 进程启动 |
| Broadcast（N 成员） | N × 5-50ms | 串行 mailbox 写入 |

### 扩展性

- 文件锁设计为 10+ 并发 agent（30 次重试，5-100ms 退避）
- In-process teammate 共享 API client 连接——不会为每个 agent 建新连接
- Mailbox 是 JSON 数组追加——大量消息时文件增长，无自动清理
- Task 认领是乐观锁——高并发下可能有重试但不会死锁
