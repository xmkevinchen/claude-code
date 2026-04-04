# Claude Code + tmux 配置指南

基于源码分析的 tmux 兼容性说明和推荐配置。

---

## Claude Code 的 tmux 感知行为

Claude Code 启动时会检测 tmux 环境（`$TMUX` 环境变量），并据此调整行为：

### 1. 全屏模式（`CLAUDE_CODE_NO_FLICKER`）

开启后进入 terminal alternate screen，使用虚拟化滚动渲染，消除输出闪烁/跳动。

| 用户类型 | 默认 |
|---------|------|
| Anthropic 内部 | 开启 |
| 外部用户 | 关闭 |

```bash
export CLAUDE_CODE_NO_FLICKER=1   # 开启
export CLAUDE_CODE_NO_FLICKER=0   # 关闭
```

### 2. 鼠标捕获

全屏模式默认开启 SGR 鼠标追踪（DEC 1000/1002/1006），Claude Code 会接管鼠标事件。这会导致：
- tmux 原生鼠标选择失效
- 选择文本时如果 Claude 继续输出，选区会被冲掉

```bash
export CLAUDE_CODE_DISABLE_MOUSE=1       # 关闭鼠标捕获，鼠标归 tmux
export CLAUDE_CODE_DISABLE_MOUSE_CLICKS=1 # 只关点击，保留滚轮
```

### 3. 颜色降级

tmux 下 Claude Code 会把 truecolor（24bit）强制降到 256 色，因为很多 tmux 配置不传递 truecolor。如果已在 tmux.conf 中配置了 `terminal-overrides` 的 `Tc` flag：

```bash
export CLAUDE_CODE_TMUX_TRUECOLOR=1   # 告诉 Claude Code 不要降级
```

### 4. `tmux -CC`（iTerm2 集成模式）

Claude Code 会自动检测 iTerm2 的 `tmux -CC` 模式并禁用全屏，因为在该模式下 alt-screen + 鼠标追踪会破坏终端状态（双击损坏、滚轮失效）。

检测逻辑：
1. 环境变量启发式：`$TMUX` + `$TERM_PROGRAM=iTerm.app` + `$TERM` 非 screen/tmux 前缀
2. 不确定时同步调用 `tmux display-message -p '#{client_control_mode}'`（~5ms）

如果要强制在 `-CC` 模式下开全屏，`CLAUDE_CODE_NO_FLICKER=1` 可以 override（不推荐）。

---

## 推荐配置

### tmux.conf 需要的项

```bash
# 鼠标支持（拖选自动进 copy-mode，冻结画面）
set -g mouse on

# truecolor 传递
set -g default-terminal "tmux-256color"
set-option -ga terminal-overrides ",xterm-256color:Tc"
```

### Shell rc 需要加的环境变量

```bash
# Claude Code + tmux 优化
export CLAUDE_CODE_NO_FLICKER=1        # 全屏防闪烁
export CLAUDE_CODE_DISABLE_MOUSE=1     # 鼠标归 tmux 管（解决选择被输出冲掉的问题）
export CLAUDE_CODE_TMUX_TRUECOLOR=1    # 已配好 Tc，不降级颜色
```

### 效果

| 操作 | 行为 |
|------|------|
| 鼠标拖选 | tmux 自动进 copy-mode，画面冻结，Claude 后台继续 |
| 松开鼠标 | tmux-yank 自动复制到系统剪贴板 |
| 滚轮 | tmux copy-mode 滚动历史 |
| 键盘滚动 | PgUp/PgDn/Ctrl+Home/End 在全屏模式下工作 |
| Claude 输出 | 全屏渲染，无闪烁 |
| 颜色 | 完整 truecolor |

---

## 不用全屏的替代方案

如果不想用全屏模式，可以靠 tmux copy-mode 手动冻结：

```bash
# 按 prefix + [ 进入 copy-mode（画面冻结）
# vi 按键选择文本
# 按 y 复制（tmux-yank）
# 按 q 退出 copy-mode
```

但这需要手动进入 copy-mode，不如 `CLAUDE_CODE_DISABLE_MOUSE=1` 自动。

---

## 所有 tmux 相关环境变量

| 变量 | 作用 | 建议值 |
|------|------|-------|
| `CLAUDE_CODE_NO_FLICKER` | 全屏防闪烁 | `1` |
| `CLAUDE_CODE_DISABLE_MOUSE` | 关闭鼠标捕获 | `1` |
| `CLAUDE_CODE_DISABLE_MOUSE_CLICKS` | 只关点击保留滚轮 | 不需要（已关全部鼠标） |
| `CLAUDE_CODE_TMUX_TRUECOLOR` | 不降级颜色 | `1`（需 tmux Tc 支持） |
| `CLAUDE_CODE_TMUX_PREFIX` | tmux 前缀键（内部用，worktree agent 设置） | 自动检测 |
| `CLAUDE_CODE_TMUX_SESSION` | tmux session 名（内部用） | 自动设置 |
| `CLAUDE_CODE_SHELL` | 自定义 shell 路径 | 通常不需要设 |
