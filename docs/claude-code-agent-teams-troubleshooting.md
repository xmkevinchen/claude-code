# Agent Teams 故障排查与防御

为什么 Team 经常删不掉、成员不退出、任务列表清不掉，以及怎么防御。

---

## 根因分析

### TeamDelete 的检查逻辑

```typescript
// TeamDeleteTool.ts:85-87
const activeMembers = nonLeadMembers.filter(m => m.isActive !== false)
if (activeMembers.length > 0) {
  return "Cannot cleanup team with N active member(s)..."
}
```

`isActive` 字段定义（`teamHelpers.ts:87`）：
```typescript
isActive?: boolean  // false when idle, undefined/true when active
```

**`undefined` 也被当作 active**。只要 config.json 中某个成员的 `isActive` 不是严格的 `false`，TeamDelete 就拒绝执行。

### 三条断裂链路

**链路 1：Shutdown 协商失败**

正常流程：
```
Leader: SendMessage({ type: "shutdown_request" }) → 写入 teammate mailbox
  → Teammate idle 轮询读到 → 模型响应 shutdown_response { approve: true }
    → AbortController.abort() → Stop hook → setMemberActive(false) → config.json 更新
```

任何一步失败 `isActive` 就留在 `undefined/true`：
- Teammate 的模型没正确响应 shutdown_request（常见——模型可能忽略或误解结构化消息）
- Teammate 正在 API 调用中，500ms 轮询没读到 mailbox
- `setMemberActive()` 的文件锁获取失败（10 次重试可能不够）
- AbortController 已经被 abort 但 Stop hook 没来得及跑

**链路 2：In-process teammate 异常退出**

```
Teammate 在 API 调用中崩溃（timeout、网络错误、unhandled rejection）
  → runAgent() 的 Promise reject
    → 如果 catch 没有调用 setMemberActive(false)
      → config.json 中 isActive 留在 undefined
```

**链路 3：Pane-based teammate 进程死亡**

```
tmux pane 中的 Claude 进程被 kill -9 / OOM / 网络断开
  → Stop hook 没有机会运行（进程已死）
    → setMemberActive(false) 没有执行
      → config.json 中 isActive 留在 true
```

### 任务列表清不掉

```
任务存储: ~/.claude/tasks/{team-name}/
TeamDelete 才会清理这个目录（TeamDeleteTool.ts:101）
TeamDelete 被 active members 阻塞 → 任务目录永远无法删除
```

### 设计缺陷总结

| 问题 | 根因 |
|------|------|
| `isActive?: boolean`，`undefined` 当 active | 缺少显式默认值 |
| Shutdown 依赖模型正确响应 | 无超时强制 abort 机制 |
| 无 force-delete | TeamDelete 没有 force 参数 |
| 进程死亡时 Stop hook 不跑 | SIGKILL 无法 trap |
| 文件锁可能阻塞 setMemberActive | 重试次数可能不够 |
| 结构化消息不能 broadcast | `to: "*"` 只接受 string，无法群发 shutdown_request |

---

## 防御手段

### 1. SessionEnd Hook（已部署）

每次 Claude Code session 结束时自动运行清理脚本，将所有成员强制标记为 inactive。

**配置**（`~/.claude/settings.json`）：
```json
{
  "hooks": {
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash /Users/ckai/.claude/scripts/team-cleanup.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

这解决了最常见的情况：session 退出时 teammate 的 Stop hook 没能正确设置 `isActive = false`。

### 2. 清理脚本

**温和清理**（`~/.claude/scripts/team-cleanup.sh`）：

将所有成员的 `isActive` 标记为 `false`，之后可以正常调用 TeamDelete。

```bash
# 清理所有 team
bash ~/.claude/scripts/team-cleanup.sh

# 清理指定 team
bash ~/.claude/scripts/team-cleanup.sh my-team
```

**核弹清理**（`~/.claude/scripts/team-nuke.sh`）：

直接删除所有 team 和 task 目录。无需经过 TeamDelete。

```bash
bash ~/.claude/scripts/team-nuke.sh
```

### 3. Session 内应急

遇到 TeamDelete 失败时，在 Claude Code 中：

```
! bash ~/.claude/scripts/team-cleanup.sh
```

`!` 前缀在 Claude Code prompt 中直接执行 shell 命令。清理完后正常调用 TeamDelete。

### 4. 新 Session 启动前检查

```bash
# 检查是否有残留 team
ls ~/.claude/teams/ 2>/dev/null

# 有残留就先清理
bash ~/.claude/scripts/team-cleanup.sh
```

可以加到 `.zshrc` 中自动执行：
```bash
# 启动 Claude Code 前自动清理残留 team
alias claude-clean='bash ~/.claude/scripts/team-cleanup.sh 2>/dev/null; claude'
```

### 5. CLAUDE.md 约定

在项目或全局 CLAUDE.md 中加入规范，引导模型正确处理 shutdown：

```markdown
## Agent Teams 关闭规范
- 关闭 team 前，对每个 teammate 单独发 shutdown_request（不能 broadcast 结构化消息）
- 每发一个 shutdown_request 后等待对应的 shutdown_response
- 如果 teammate 没有响应，直接运行 `bash ~/.claude/scripts/team-cleanup.sh`
- 然后调用 TeamDelete 清理
```

### 6. 手动编辑 config.json

如果脚本不可用，直接编辑：

```bash
# 找到 team 配置
cat ~/.claude/teams/*/config.json | jq '.name, [.members[] | {name, isActive}]'

# 强制标记所有成员为 inactive
TEAM="your-team-name"
jq '.members |= map(.isActive = false)' \
  ~/.claude/teams/$TEAM/config.json > /tmp/team.json \
  && mv /tmp/team.json ~/.claude/teams/$TEAM/config.json
```

### 7. 手动删除目录

最后手段：

```bash
# 删除特定 team
rm -rf ~/.claude/teams/my-team ~/.claude/tasks/my-team

# 删除所有 team（核弹）
rm -rf ~/.claude/teams/ ~/.claude/tasks/
```

注意：这只删除磁盘状态。如果 Claude Code session 还在运行，`AppState.teamContext` 仍然存在于内存中。需要重启 session。

---

## 防御矩阵

| 场景 | 防御手段 | 自动/手动 |
|------|---------|----------|
| 正常 session 退出 | SessionEnd hook | 自动 |
| 异常退出（crash/kill） | 下次启动前 cleanup | 需手动或 alias |
| 模型没响应 shutdown | cleanup 脚本 + TeamDelete | 手动 |
| Pane-based teammate 进程死亡 | cleanup 脚本 | 手动 |
| 任务列表残留 | cleanup → TeamDelete（会清理 tasks） | 手动 |
| 完全卡死无法操作 | nuke 脚本 | 手动 |
| 内存中 teamContext 残留 | 重启 Claude Code | 手动 |

---

## 文件位置

| 文件 | 用途 |
|------|------|
| `~/.claude/scripts/team-cleanup.sh` | 温和清理（标记 inactive） |
| `~/.claude/scripts/team-nuke.sh` | 核弹清理（删除目录） |
| `~/.claude/teams/{name}/config.json` | Team 配置（成员 isActive 状态在这里） |
| `~/.claude/teams/{name}/inboxes/{agent}.json` | Teammate mailbox |
| `~/.claude/tasks/{name}/{id}.json` | 共享任务列表 |
| `~/.claude/settings.json` → `hooks.SessionEnd` | 自动清理 hook 配置 |
