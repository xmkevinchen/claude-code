# Claude Code Feature Flags & Environment Variables 完整清单

基于源码全量搜索。

---

## 概要

| 类别 | 数量 |
|------|------|
| `feature()` 构建时开关 | **96 个** |
| `CLAUDE_CODE_*` 环境变量 | **196 个** |
| `CLAUDE_*` 环境变量（非 CODE） | **20 个** |
| `DISABLE_*` 环境变量 | **22 个** |
| `ENABLE_*` 环境变量 | **11 个** |

关键区分：`feature()` 是构建时死代码消除，外部构建中被禁用的无法通过任何方式激活。环境变量绝大多数是运行时的，直接可用。

---

## 一、96 个 Feature Flags（构建时开关）

按功能域分类：

### 自主代理 / KAIROS
| Flag | 用途 |
|------|------|
| `KAIROS` | 持久自主代理身份 |
| `KAIROS_BRIEF` | 简报模式 UI |
| `KAIROS_CHANNELS` | Channel-based MCP 通知 |
| `KAIROS_DREAM` | 后台记忆整合 |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub webhook 订阅（SubscribePRTool） |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知 |
| `PROACTIVE` | 主动执行模式 |
| `DAEMON` | 后台守护进程 |

### 多代理协作
| Flag | 用途 |
|------|------|
| `COORDINATOR_MODE` | 协调者/工人拓扑 |
| `FORK_SUBAGENT` | Agent 在独立 git worktree 运行 |
| `TEAMMEM` | 团队共享记忆 |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 内置 Explore/Plan 专用 agent |
| `VERIFICATION_AGENT` | 任务完成后验证 agent |
| `AGENT_MEMORY_SNAPSHOT` | Agent 记忆快照 |

### 定时任务 / 后台
| Flag | 用途 |
|------|------|
| `AGENT_TRIGGERS` | Cron 定时任务（CronCreate 等） |
| `AGENT_TRIGGERS_REMOTE` | 远程定时代理 |
| `BG_SESSIONS` | 后台 session（--bg） |

### Context / 压缩
| Flag | 用途 |
|------|------|
| `CACHED_MICROCOMPACT` | Cache-editing beta 内联压缩 |
| `CONTEXT_COLLAPSE` | 替代 autocompact 的渐进式 context 提交 |
| `REACTIVE_COMPACT` | 仅在 API 返回 413 后才压缩 |
| `HISTORY_SNIP` | 选择性消息修剪 |
| `COMPACTION_REMINDERS` | 压缩提醒 |

### 远程 / Bridge
| Flag | 用途 |
|------|------|
| `BRIDGE_MODE` | 远程 session 基础设施 |
| `CCR_AUTO_CONNECT` | CCR 自动连接（ant-only） |
| `CCR_MIRROR` | CCR 镜像 |
| `CCR_REMOTE_SETUP` | 远程环境配置 |
| `SSH_REMOTE` | SSH 远程 |
| `DIRECT_CONNECT` | 直连模式 |

### 安全 / 分类器
| Flag | 用途 |
|------|------|
| `TRANSCRIPT_CLASSIFIER` | 自动模式安全分类器 |
| `BASH_CLASSIFIER` | Bash 命令安全分类器 |
| `ANTI_DISTILLATION_CC` | 注入 fake_tools 抵抗蒸馏 |
| `NATIVE_CLIENT_ATTESTATION` | 客户端原生认证 |

### 工具
| Flag | 用途 |
|------|------|
| `CHICAGO_MCP` | Computer-use MCP 集成 |
| `WEB_BROWSER_TOOL` | 无头浏览器工具 |
| `MONITOR_TOOL` | 后台监控工具 |
| `TERMINAL_PANEL` | 嵌入式终端面板 |
| `OVERFLOW_TEST_TOOL` | 内部测试工具 |
| `WORKFLOW_SCRIPTS` | 捆绑工作流脚本 |
| `TOKEN_BUDGET` | 任务级 token 预算 |
| `ULTRAPLAN` | 远程 Opus 深度规划 |
| `ULTRATHINK` | 扩展思考模式 |

### MCP / 技能
| Flag | 用途 |
|------|------|
| `MCP_SKILLS` | MCP server 暴露 skills |
| `MCP_RICH_OUTPUT` | MCP 富文本输出 |
| `EXPERIMENTAL_SKILL_SEARCH` | 实验性技能搜索 |
| `RUN_SKILL_GENERATOR` | 技能生成器 |
| `SKILL_IMPROVEMENT` | 技能改进 |

### UI / 终端
| Flag | 用途 |
|------|------|
| `VOICE_MODE` | 语音输入 |
| `BUDDY` | Tamagotchi 宠物系统 |
| `AUTO_THEME` | 自动主题 |
| `QUICK_SEARCH` | 快速搜索 |
| `MESSAGE_ACTIONS` | 消息操作 |
| `STREAMLINED_OUTPUT` | 精简输出 |
| `REVIEW_ARTIFACT` | 审查工件 |
| `HISTORY_PICKER` | 历史选择器 |
| `SHOT_STATS` | 统计数据 |

### 记忆
| Flag | 用途 |
|------|------|
| `EXTRACT_MEMORIES` | 自动提取记忆 |
| `MEMORY_SHAPE_TELEMETRY` | 记忆形态遥测 |
| `FILE_PERSISTENCE` | 文件持久化 |

### 构建 / 内部
| Flag | 用途 |
|------|------|
| `ABLATION_BASELINE` | 消融测试基线 |
| `ALLOW_TEST_VERSIONS` | 允许测试版本 |
| `BUILDING_CLAUDE_APPS` | 构建 Claude 应用引导 |
| `BYOC_ENVIRONMENT_RUNNER` | BYOC 环境运行器 |
| `COMMIT_ATTRIBUTION` | 提交归属 |
| `CONNECTOR_TEXT` | 连接器文本 |
| `COWORKER_TYPE_TELEMETRY` | 协作者类型遥测 |
| `DUMP_SYSTEM_PROMPT` | 导出系统 prompt |
| `ENHANCED_TELEMETRY_BETA` | 增强遥测 beta |
| `HARD_FAIL` | 硬失败模式 |
| `HOOK_PROMPTS` | Hook prompt 注入 |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | libc 检测 |
| `LODESTONE` | 用途不明 |
| `NEW_INIT` | 新初始化流程 |
| `PERFETTO_TRACING` | Perfetto 性能追踪 |
| `POWERSHELL_AUTO_MODE` | PowerShell 自动模式 |
| `PROMPT_CACHE_BREAK_DETECTION` | Prompt cache break 检测 |
| `SELF_HOSTED_RUNNER` | 自托管运行器 |
| `SKIP_DETECTION_WHEN_AUTOUPDATES_DISABLED` | 跳过检测 |
| `SLOW_OPERATION_LOGGING` | 慢操作日志 |
| `TEMPLATES` | 任务模板 |
| `TORCH` | 用途不明 |
| `TREE_SITTER_BASH` / `TREE_SITTER_BASH_SHADOW` | Bash tree-sitter 解析 |
| `UDS_INBOX` | Unix domain socket 消息 |
| `UNATTENDED_RETRY` | 无人值守重试 |
| `DOWNLOAD_USER_SETTINGS` / `UPLOAD_USER_SETTINGS` | 云端设置同步 |
| `BREAK_CACHE_COMMAND` | 缓存刷新命令 |

---

## 二、外部用户可用的环境变量

以下环境变量**不在 feature() 保护内**，直接可用。

### 模式控制

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_SIMPLE=1` | 精简模式：只保留 Bash+FileRead+FileEdit，跳过 agent/skill/hook 加载 | 未设置 |
| `CLAUDE_CODE_COORDINATOR_MODE=1` | 协调者模式（需构建支持） | 未设置 |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | Agent Teams 实验特性 | 未设置 |

### 成本控制

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_DISABLE_FAST_MODE=1` | 关闭 fast mode（6x 省钱） | 未设置 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=N` | 压缩触发阈值 % | 模型相关 |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW=N` | 有效 context 大小上限 | 模型上限 |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS=N` | 最大 context token 数 | 模型上限 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS=N` | 最大 output token 数 | 8000 起 |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=N` | 工具并发上限 | 10 |
| `CLAUDE_CODE_EFFORT_LEVEL=low/medium/high` | 推理努力程度 | 未设置 |
| `CLAUDE_CODE_IDLE_THRESHOLD_MINUTES=N` | 空闲阈值（分钟） | - |
| `CLAUDE_CODE_IDLE_TOKEN_THRESHOLD=N` | 空闲 token 阈值 | - |

### 压缩控制

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `DISABLE_AUTO_COMPACT=1` | 禁用自动压缩（保留手动 /compact） | 未设置 |
| `DISABLE_COMPACT=1` | 禁用所有压缩 | 未设置 |
| `DISABLE_CLAUDE_CODE_SM_COMPACT=1` | 禁用 session memory 压缩 | 未设置 |
| `ENABLE_CLAUDE_CODE_SM_COMPACT=1` | 启用 session memory 压缩 | 未设置 |

### Prompt Cache 控制

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `DISABLE_PROMPT_CACHING=1` | 全局禁用 prompt cache | 未设置 |
| `DISABLE_PROMPT_CACHING_HAIKU=1` | Haiku 禁用 cache | 未设置 |
| `DISABLE_PROMPT_CACHING_SONNET=1` | Sonnet 禁用 cache | 未设置 |
| `DISABLE_PROMPT_CACHING_OPUS=1` | Opus 禁用 cache | 未设置 |
| `ENABLE_PROMPT_CACHING_1H_BEDROCK=1` | Bedrock 1 小时 TTL cache | 未设置 |

### API / Provider 配置

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_API_BASE_URL=url` | 自定义 API 端点 | Anthropic 默认 |
| `CLAUDE_CODE_USE_BEDROCK=1` | 使用 AWS Bedrock | 未设置 |
| `CLAUDE_CODE_USE_VERTEX=1` | 使用 Google Vertex AI | 未设置 |
| `CLAUDE_CODE_USE_FOUNDRY=1` | 使用 Foundry | 未设置 |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH=1` | 跳过 Bedrock 认证检查 | 未设置 |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH=1` | 跳过 Vertex 认证检查 | 未设置 |
| `CLAUDE_CODE_SUBAGENT_MODEL=model` | 子 agent 默认模型 | 继承父级 |
| `CLAUDE_CODE_AUTO_MODE_MODEL=model` | 自动模式分类器模型 | - |
| `CLAUDE_CODE_MAX_RETRIES=N` | API 最大重试次数 | - |

### 功能开关

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_DISABLE_THINKING=1` | 禁用扩展思考 | 未设置 |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` | 禁用自适应思考 | 未设置 |
| `CLAUDE_CODE_DISABLE_ATTACHMENTS=1` | 禁用附件 | 未设置 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` | 禁用后台任务 | 未设置 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | 禁用自动记忆 | 未设置 |
| `CLAUDE_CODE_DISABLE_CRON=1` | 禁用定时任务 | 未设置 |
| `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS=1` | 禁用 git 指令注入 | 未设置 |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` | 禁用 CLAUDE.md 发现 | 未设置 |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING=1` | 禁用文件检查点 | 未设置 |
| `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK=1` | 禁用命令注入检查 | 未设置 |
| `CLAUDE_CODE_DISABLE_MOUSE=1` | 禁用鼠标支持 | 未设置 |
| `CLAUDE_CODE_DISABLE_ADVISOR_TOOL=1` | 禁用 advisor 工具 | 未设置 |
| `DISABLE_INTERLEAVED_THINKING=1` | 禁用交错思考 | 未设置 |
| `DISABLE_COST_WARNINGS=1` | 禁用成本警告 | 未设置 |
| `DISABLE_TELEMETRY=1` | 禁用遥测 | 未设置 |
| `ENABLE_LSP_TOOL=1` | 启用 LSP 工具 | 未设置 |
| `ENABLE_TOOL_SEARCH=1` | 启用工具搜索 | 未设置 |

### 终端 / UI

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_SHELL=path` | 自定义 shell 路径 | 系统默认 |
| `CLAUDE_CODE_SHELL_PREFIX=cmd` | Shell 命令前缀 | - |
| `CLAUDE_CODE_TMUX_PREFIX=key` | tmux 前缀键 | - |
| `CLAUDE_CODE_TMUX_TRUECOLOR=1` | tmux 真彩色 | 未设置 |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT=1/0` | 语法高亮 | 开启 |
| `CLAUDE_CODE_SCROLL_SPEED=N` | 滚动速度 | - |
| `CLAUDE_CODE_NO_FLICKER=1` | 防闪烁 | 未设置 |
| `CLAUDE_CODE_ACCESSIBILITY=1` | 无障碍模式 | 未设置 |
| `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL=1` | 禁用虚拟滚动 | 未设置 |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1` | 禁用终端标题修改 | 未设置 |

### 文件操作

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS=N` | 文件读取最大 token | - |
| `CLAUDE_CODE_GLOB_HIDDEN=1` | Glob 包含隐藏文件 | 未设置 |
| `CLAUDE_CODE_GLOB_NO_IGNORE=1` | Glob 忽略 .gitignore | 未设置 |
| `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS=N` | Glob 超时秒数 | - |

### 调试 / 诊断

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_DEBUG_LOG_LEVEL=level` | 调试日志级别 | - |
| `CLAUDE_CODE_DEBUG_LOGS_DIR=path` | 调试日志目录 | - |
| `CLAUDE_CODE_DIAGNOSTICS_FILE=path` | 诊断文件路径 | - |
| `CLAUDE_CODE_JSONL_TRANSCRIPT=path` | JSONL 转录路径 | 未设置 |
| `CLAUDE_CODE_PROFILE_STARTUP=1` | 启动性能分析 | 未设置 |
| `CLAUDE_CODE_PROFILE_QUERY=1` | 查询性能分析 | 未设置 |
| `CLAUDE_DEBUG=1` | 通用调试模式 | 未设置 |
| `DISABLE_ERROR_REPORTING=1` | 禁用错误上报 | 未设置 |

### 认证

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_OAUTH_TOKEN=token` | OAuth token | - |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL=url` | 自定义 OAuth URL | - |
| `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR=fd` | API key 文件描述符 | - |
| `CLAUDE_CODE_CLIENT_CERT=path` | 客户端证书 | - |
| `CLAUDE_CODE_CLIENT_KEY=path` | 客户端密钥 | - |

### 插件

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_PLUGIN_CACHE_DIR=path` | 插件缓存目录 | - |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL=1` | 同步安装插件 | 未设置 |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS=N` | 同步安装超时 | - |

### 杂项

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_TMPDIR=path` | 临时文件目录 | 系统默认 |
| `CLAUDE_CONFIG_DIR=path` | 配置目录 | ~/.claude |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=paths` | 额外 CLAUDE.md 搜索路径 | - |
| `CLAUDE_CODE_TAGS=json` | 会话标签 | - |
| `CLAUDE_CODE_EXTRA_BODY=json` | API 请求额外 body | - |
| `CLAUDE_CODE_EXTRA_METADATA=json` | 额外元数据 | - |
