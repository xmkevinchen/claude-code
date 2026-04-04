# Claude Code 插件与技能系统

基于源码分析的完整机制说明。

---

## 概览

Claude Code 有两套扩展机制：**Plugin**（重量级）和 **Skill**（轻量级）。

| | Plugin | Skill |
|---|--------|-------|
| 包含内容 | commands、agents、skills、hooks、MCP/LSP servers | 单个 markdown 文件 |
| 有依赖系统 | 有（插件间依赖） | 没有 |
| 有版本管理 | 有（semver / git SHA） | 没有 |
| 有自动更新 | 有（marketplace 级别） | 没有 |
| 安装方式 | marketplace / 手动 | 放文件到目录 |
| 存储位置 | `~/.claude/plugins/` | `.claude/skills/` 或 `~/.claude/skills/` |
| 启用/禁用 | `settings.json` 按项目控制 | 始终可用 |

---

## 一、Plugin 系统

### 1.1 插件目录结构

```
my-plugin/
├── plugin.json              # 清单文件
├── .mcp.json                # MCP server 配置（可选）
├── commands/                # 斜杠命令（.md 文件）
├── agents/                  # AI agent 定义（.md 文件）
├── skills/                  # 技能目录
├── hooks/                   # Hook 配置
│   └── hooks.json
└── output-styles/           # 输出格式化样式
```

### 1.2 plugin.json 清单格式

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "插件描述",
  "author": { "name": "Author", "url": "https://..." },
  "license": "MIT",
  "keywords": ["tag1", "tag2"],

  "commands": "commands/",
  "agents": "agents/",
  "skills": "skills/",
  "hooks": "hooks/hooks.json",
  "outputStyles": "output-styles/",

  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["server.js"],
      "env": { "API_KEY": "${userConfig.apiKey}" }
    }
  },

  "lspServers": {
    "my-lsp": {
      "command": "my-lsp-binary",
      "args": ["--stdio"]
    }
  },

  "dependencies": [
    "other-plugin",
    "cross-market-plugin@other-marketplace"
  ],

  "userConfig": {
    "apiKey": {
      "type": "string",
      "description": "API key for the service",
      "required": true
    }
  }
}
```

### 1.3 依赖系统

**声明格式**：

```json
{
  "dependencies": [
    "auth-helper",                     // 同 marketplace 内，自动继承
    "ui-lib@other-marketplace"         // 跨 marketplace（需白名单）
  ]
}
```

**解析机制**（`dependencyResolver.ts`）：
- DFS 遍历依赖树
- 循环依赖检测
- 跨 marketplace 依赖默认禁止（安全边界）
- 根 marketplace 的 `allowCrossMarketplaceDependenciesOn` 白名单可放行特定跨 marketplace 依赖

**加载时验证**（`verifyAndDemote()`）：
- 检查所有已启用插件的依赖是否也已启用
- 依赖未满足的插件被自动降级（fixed-point 循环直到稳定）
- 不会静默失败——降级的插件会在 `/plugin` UI 中显示警告

**限制**：依赖只能是其他 Claude Code 插件。不能声明系统包（npm、pip、apt 等）。

### 1.4 版本管理

**版本来源**（优先级递减）：

1. `plugin.json` 的 `version` 字段（显式声明）
2. Marketplace 条目中的版本
3. Git commit SHA（完整 40 字符截断为 12）
4. Git 子目录：SHA + 路径哈希（monorepo 防碰撞）
5. Fallback: `'unknown'`

**缓存路径**：每个版本独立目录，支持回滚。

```
~/.claude/plugins/cache/{marketplace}/{plugin-name}/{version}/
```

### 1.5 自动更新

**触发时机**：每次 Claude Code 启动时（后台非阻塞）。

**流程**：
1. 检查已声明 marketplace 的 `autoUpdate` 设置
2. 拉取最新 marketplace 定义
3. 比较已安装插件版本
4. 有更新则下载到新版本目录
5. 下次 session 生效（当前 session 使用内存快照）

**默认行为**：
- 官方 Anthropic marketplace：自动更新默认开启
- 其他 marketplace：可配置 `extraKnownMarketplaces[name].autoUpdate = false`
- 回调通知：`onPluginsAutoUpdated()` 在 REPL UI 显示更新完成

### 1.6 Marketplace 系统

**Marketplace 来源类型**：

| 类型 | 格式 | 示例 |
|------|------|------|
| GitHub | `{ source: "github", repo: "owner/repo" }` | `anthropics/claude-plugins-official` |
| Git | `{ source: "git", url: "..." }` | 任意 git 仓库 |
| URL | HTTPS 链接指向 `marketplace.json` | CDN 托管 |
| NPM | npm 包名 | `@org/marketplace` |
| 本地目录 | `{ source: "local", path: "/path" }` | 开发调试 |
| Settings 内联 | `settings.json` 中声明 | 简单场景 |

**官方 Marketplace**：
- 主源：GCS 镜像 `https://downloads.claude.ai/claude-code-releases/plugins/claude-plugins-official/`（content-addressed zip，快速启动）
- 备源：GitHub clone `anthropics/claude-plugins-official`
- 自动更新：默认开启

**Marketplace 定义格式**（`.claude-plugin/marketplace.json`）：

```json
{
  "name": "my-marketplace",
  "description": "描述",
  "plugins": [
    {
      "name": "plugin-a",
      "description": "...",
      "source": "./plugins/plugin-a",
      "version": "1.0.0"
    },
    {
      "name": "plugin-b",
      "source": {
        "source": "github",
        "repo": "owner/repo",
        "ref": "v2.0",
        "path": "packages/plugin-b"
      }
    }
  ]
}
```

**插件源类型**（plugin 在 marketplace 中的 source）：

| 类型 | 说明 |
|------|------|
| 相对路径 | `"./path"` — 相对于 marketplace 根目录 |
| npm | `{ source: "npm", package: "@org/pkg", version: "^1.0" }` |
| npm-subdir | `{ source: "npm-subdir", package: "pkg", path: "dist/plugin" }` |
| git | `{ source: "git", url: "...", ref: "main", path: "subdir" }` |
| git-subdir | `{ source: "git-subdir", url: "...", path: "subdirs/plugin" }` |
| github | `{ source: "github", repo: "owner/repo", ref: "v1.0" }` |
| python | `{ source: "python", package: "pkg", version: ">=1.0" }` |
| local | `{ source: "local", path: "/absolute/path" }` |

### 1.7 存储与状态

**双层状态分离**：

```
安装状态（全局）：~/.claude/plugins/installed_plugins.json
启用状态（按项目）：.claude/settings.json 或 .claude/settings.local.json
```

**installed_plugins.json**（V2 格式）：

```json
{
  "version": 2,
  "plugins": {
    "plugin-name@marketplace": [
      {
        "installPath": "/full/path/to/plugin",
        "version": "1.2.3",
        "scope": "user",
        "timestamp": "2026-01-01T00:00:00Z",
        "sha": "abc123def456"
      }
    ]
  }
}
```

**启用/禁用**：

```json
// .claude/settings.local.json
{
  "enabledPlugins": {
    "ae@agentic-engineering": true,
    "another@marketplace": false
  }
}
```

**作用域**：

| Scope | 含义 | 可见范围 |
|-------|------|---------|
| `user` | 用户级安装 | 所有项目 |
| `project` | 项目级安装 | 特定项目 |
| `local` | 本地路径 | 当前机器 |
| `managed` | 策略推送 | 企业管理 |

### 1.8 安全机制

**Marketplace 防伪**：
- 保留名称：`claude-plugins-official`、`anthropic-marketplace` 等——只允许来自 `anthropics` GitHub org
- 模式拦截：匹配 `/(official.*anthropic|anthropic.*official)/i` 的名称被阻止
- 同形字保护：marketplace 名称禁止非 ASCII 字符

**策略控制**：
- `strictKnownMarketplaces`：白名单模式，只允许声明的 marketplace
- `blockedMarketplaces`：显式黑名单
- `blocklistUrl`：远程黑名单 URL

**依赖安全**：
- 跨 marketplace 自动安装默认禁止
- 已启用的依赖不会被重复安装

### 1.9 内置插件

内置插件编译在 CLI 二进制中，ID 格式 `{name}@builtin`。

```typescript
registerBuiltinPlugin({
  name: "plugin-name",
  description: "...",
  skills: [...],
  hooks: {...},
  mcpServers: {...},
  isAvailable: () => boolean,  // 条件可用性
  defaultEnabled: true
})
```

用户可在 `/plugin` UI 中启用/禁用内置插件。

---

## 二、Skill 系统

### 2.1 技能来源（优先级递减）

1. **Bundled**：编译在 CLI 中（`src/skills/bundled/`）
2. **Plugin**：来自已启用插件的 `commands/` 目录
3. **Managed**：企业策略推送
4. **MCP**：MCP server 暴露的 resource-as-skill（`feature('MCP_SKILLS')`）
5. **Project**：`.claude/skills/*.md`
6. **User**：`~/.claude/skills/*.md`

### 2.2 技能 Frontmatter 格式

```yaml
---
name: "my-skill"
description: "一句话描述"
when_to_use: "Use when the user wants to..."
argument-hint: "[file] [options]"
allowed-tools:
  - Read
  - Grep
  - "Bash(git:*)"
arguments:
  - arg1
  - arg2
version: "1.0.0"
model: "claude-opus"          # 模型覆盖（或 'inherit'）
user-invocable: true          # false 则不在 UI 中显示
effort: "medium"              # low / medium / high / 1-10
context: "fork"               # fork（独立 context）或 inline（共享 context）
agent: "agent-name"           # 指定执行 agent
paths: "**/*.ts"              # 路径过滤
---

# Skill 内容

这里是 skill 的 prompt 内容，支持 Markdown。
```

### 2.3 内置技能列表

| 技能 | 用途 | 用户可调用 |
|------|------|-----------|
| `update-config` | 配置 settings.json | 是 |
| `keybindings` | 配置键盘快捷键 | 是 |
| `verify` | 验证任务完成情况 | 是 |
| `debug` | 调试辅助 | 是 |
| `simplify` | 代码简化审查 | 是 |
| `remember` | 保存记忆 | 是 |
| `batch` | 批量操作 | 是 |
| `stuck` | 卡住时的辅助 | 是 |
| `skillify` | 创建新技能 | 是 |
| `loop` | 定时循环执行 | 是（需 `AGENT_TRIGGERS`） |
| `schedule` | 远程定时代理 | 是（需 `AGENT_TRIGGERS_REMOTE`） |
| `dream` | 记忆整合 | 是（需 `KAIROS_DREAM`） |
| `claude-api` | Claude API 开发辅助 | 是 |

### 2.4 技能去重与发现

- 通过 symlink 或重叠目录访问的同一文件会去重（`realpath()` 规范化）
- 先发现的技能保留，后续重复跳过
- Token 预估只计算 frontmatter（name + description + when_to_use），完整内容在调用时才加载

---

## 三、MCP Server 与插件的关系

### 声明方式

**在 plugin.json 中**：

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["server.js"],
      "env": { "API_KEY": "${userConfig.apiKey}" }
    }
  }
}
```

**或外部文件**：

```json
{
  "mcpServers": "./mcp.json"
}
```

**或 MCPB/DXT 格式**：

```json
{
  "mcpServers": "./servers.mcpb"
}
```

### userConfig 与 MCP

插件的 `userConfig` 字段可以声明用户需要填写的配置项（如 API key）。MCP server 的 `env` 字段通过 `${userConfig.xxx}` 引用这些值。启用插件时，如果有未配置的 required userConfig，插件状态为 `needs-config`，用户被提示填写后才能启动 MCP server。

---

## 四、依赖安装的限制

插件依赖系统**只管理 Claude Code 插件之间的依赖**。不能声明或自动安装：

- npm packages（`node_modules`）
- pip packages
- 系统二进制（apt、brew）
- 其他运行时依赖

### 变通方案

| 场景 | 做法 |
|------|------|
| MCP server 需要 npm 包 | `command: "npx"`, `args: ["-y", "package-name"]`（npx 临时安装） |
| 需要 Python 依赖 | `command: "uvx"`, `args: ["package-name"]`（uvx 临时安装） |
| 需要编译好的二进制 | 插件自带二进制或在 marketplace 构建步骤中预编译 |
| 复杂安装流程 | 在插件 README 中说明手动安装步骤 |

类似 VS Code 扩展——扩展间有依赖管理，但不管系统级包安装。

---

## 五、文件位置总览

```
~/.claude/
├── plugins/
│   ├── installed_plugins.json          # 安装元数据（V2）
│   ├── known_marketplaces.json         # 已声明 marketplace 列表
│   ├── cache/                          # 插件缓存
│   │   └── {marketplace}/{plugin}/{version}/
│   ├── marketplaces/                   # Marketplace 本地克隆/缓存
│   │   ├── claude-plugins-official/    # 官方 marketplace
│   │   └── {other-marketplace}/
│   └── plugin-data/                    # 插件私有数据
│       └── {marketplace}/{plugin}/
├── skills/                             # 用户全局技能
│   └── my-skill.md
├── agents/                             # 用户全局 agent
└── settings.json                       # 全局设置（含权限、插件启用）

{project}/
├── .claude/
│   ├── settings.local.json             # 项目级插件启用
│   ├── skills/                         # 项目级技能
│   └── agents/                         # 项目级 agent
└── ...
```
