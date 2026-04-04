# Claude Code 省钱指南

基于源码分析的实操技巧，适用于 API key 用户和 Claude.ai 订阅用户。

---

## 一、最高优先级：关掉 Fast Mode（6x 差价）

| 模式 | Input / MTok | Output / MTok |
|------|-------------|---------------|
| Opus 4.6 标准 | $5 | $25 |
| Opus 4.6 Fast | $30 | $150 |
| Sonnet 4.6 | $3 | $15 |
| Haiku 4.5 | $1 | $5 |

Fast Mode 是同一个 Opus 4.6 模型的加速输出，不是更好的模型。对于非紧急任务，标准速度足够。

```bash
# 强制关闭 fast mode
export CLAUDE_CODE_DISABLE_FAST_MODE=1
```

API key 用户默认关闭，无需操作。Claude.ai 订阅用户默认可能开启，建议检查。

---

## 二、利用 Prompt Cache（10x 差价）

Cache read 比正常 input token 便宜 10 倍：

| 模型 | Input / MTok | Cache Read / MTok | 节省 |
|------|-------------|-------------------|------|
| Opus 4.6 标准 | $15 | $1.50 | 10x |
| Opus 4.6 Fast | $30 | $3.00 | 10x |
| Sonnet 4.6 | $3 | $0.30 | 10x |
| Haiku 4.5 | $0.80 | $0.08 | 10x |

### 怎么最大化 cache 命中率

**1. 保持 CLAUDE.md 内容丰富且稳定**

CLAUDE.md 被 cache-write 一次后，后续每轮对话都是 cache-read。把你常用的项目上下文、约定、架构说明写进去，这些内容每次 turn 都"免费"重读。

**2. 不要频繁增减 MCP 工具**

源码追踪了 15 个维度的 cache break 原因（`promptCacheBreakDetection.ts`）。工具 schema 变化是最常见的 cache bust 原因（占 77%）。建议：
- 一次性配好需要的 MCP server，不要每个 session 装卸
- 如果临时不需要某个 MCP server，考虑保留而非移除

**3. Bedrock 用户开启 1 小时 TTL cache**

```bash
export ENABLE_PROMPT_CACHING_1H_BEDROCK=1
```

默认 cache TTL 是 5 分钟。1 小时 TTL 对长 session 或多次短 session 场景效果显著。

**4. 调试/开发 prompt 时可临时禁用 cache**

```bash
export DISABLE_PROMPT_CACHING=1          # 全局禁用
export DISABLE_PROMPT_CACHING_HAIKU=1    # 按模型禁用
export DISABLE_PROMPT_CACHING_SONNET=1
export DISABLE_PROMPT_CACHING_OPUS=1
```

---

## 三、子任务选对模型

在 Agent 工具调用中可以指定 `model` 参数：

| 任务类型 | 推荐模型 | 理由 |
|---------|---------|------|
| 搜索/grep/文件发现 | `haiku` | 搜索密集型，不需要深度推理，$1/$5 vs Opus $15/$75 |
| 代码审查/架构分析 | `sonnet` | 平衡质量和成本 |
| 复杂推理/设计决策 | `opus`（默认） | 需要最强推理能力时才用 |

同理，`reasoning effort` 参数（high/medium/low）可以在不切换模型的前提下控制推理深度和成本。

---

## 四、控制 Context Window 使用

### 精简工具集

```bash
export CLAUDE_CODE_SIMPLE=1
```

只保留 `Bash` + `FileRead` + `FileEdit` 三个工具。更少工具 = 更小 system prompt = 更少 token。适合只做简单编辑的场景。

### 控制工具并发数

```bash
export CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=3  # 默认 10
```

降低并发减少单轮 token 峰值。适合想精细控制花费的场景。

### 工具结果自动落盘

单个工具结果超过 50K 字符时自动写入磁盘，模型只拿到文件路径 + 摘要。单轮所有工具结果合计超过 200K 字符时，最大的结果优先落盘。这是自动的，无需配置。

---

## 五、调整 Autocompact 策略

Autocompact 在 context 接近满时自动触发，fork 一个子 agent 生成摘要。

```bash
# 调整触发阈值（默认值取决于模型，通常 80-90%）
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70   # 70% 时触发

# 限制有效 context window（强制更早压缩，适合小模型）
export CLAUDE_CODE_AUTO_COMPACT_WINDOW=100000

# 禁用自动压缩（保留手动 /compact 命令）
export DISABLE_AUTO_COMPACT=1

# 完全禁用所有压缩
export DISABLE_COMPACT=1
```

**建议**：
- 不要做大量极短 session——每个新 session 的 cache warm-up 和可能的压缩都有开销
- 长 session 中适时手动 `/compact` 比等自动触发更可控
- 连续压缩失败 3 次后会自动停止（断路器），不用担心死循环烧钱

---

## 六、Output Token 的隐藏优化

默认每次请求只保留 8K output token（`CAPPED_DEFAULT_MAX_TOKENS`）。实际 p99 只用 ~5K token。只有当模型确实需要更多输出时才自动升级到 64K。这是自动的，解释了为什么大多数请求比预期便宜。

避免在 prompt 中要求"详细解释每一步"——这会推高 output token 消耗。需要详细输出时再要求。

---

## 七、Token Budget 硬限额（Beta）

```bash
# 通过 beta header 启用 task budget
# task-budgets-2026-03-13
```

启用后可以设置 `maxBudgetUsd` 对每个任务硬性限制 API 花费。到达上限时任务停止而非继续烧钱。

---

## 八、成本追踪注意事项

### 已追踪
- Claude API 调用的 input/output/cache token
- 按模型分组的用量显示
- Advisor（二级模型）用量

### 未追踪
- MCP 工具在外部服务器触发的模型调用成本
- Session resume 时如果 session ID 不匹配，累计成本会静默清零
- 显示精度：> $0.50 只显示 2 位小数

### 实操建议
- 定期用 `/cost` 查看当前 session 花费
- 大型 agentic 任务结束后检查总成本
- 如果使用了会调用外部 LLM 的 MCP server，需要单独追踪那部分成本

---

## 环境变量速查表

| 变量 | 作用 | 默认值 |
|------|------|-------|
| `CLAUDE_CODE_DISABLE_FAST_MODE=1` | 关闭 fast mode | 未设置（fast mode 可用） |
| `CLAUDE_CODE_SIMPLE=1` | 精简工具集 | 未设置 |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=N` | 工具并发上限 | 10 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=N` | 压缩触发阈值 % | 模型相关 |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW=N` | 有效 context 大小上限 | 模型上限 |
| `DISABLE_AUTO_COMPACT=1` | 禁用自动压缩 | 未设置 |
| `DISABLE_COMPACT=1` | 禁用所有压缩 | 未设置 |
| `DISABLE_PROMPT_CACHING=1` | 禁用 prompt cache | 未设置 |
| `ENABLE_PROMPT_CACHING_1H_BEDROCK=1` | Bedrock 1h cache | 未设置 |
