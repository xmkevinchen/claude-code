---
id: "03"
title: "Model-agnostic 适配层"
status: pending
current_round: 0
created: 2026-04-04
decision: ""
rationale: ""
reversibility: ""
---

# Topic: Model-agnostic 适配层

## Current Status
待讨论。

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
Claude Code 深度绑定 Anthropic API——system prompt 格式、tool_use 格式、cache_control、prompt caching、extended thinking、beta headers 全部是 Claude 特有的。OpenClaw 需要支持 OpenAI、Gemini、本地模型（Ollama/vLLM）等多种 provider。

已有的多 provider 方案：LiteLLM（Python proxy）、Vercel AI SDK（TS）、OpenRouter（API proxy）。

## Constraints
- 不同模型的 tool calling 格式不同（Claude tool_use vs OpenAI function calling vs Gemini function declarations）
- 不同模型的 context window、token 计费、streaming 格式不同
- Prompt caching 是 Claude 特有的（OpenAI 自动 cache，Gemini 有 context caching）
- Extended thinking 是 Claude 特有的
- 需要在不牺牲太多性能的前提下做抽象

## Key Questions
- 适配层放在哪？内部抽象（每个 provider 一个 adapter）还是外部 proxy（LiteLLM）？
- Tool calling 格式转换：统一到哪个格式？还是每个 provider 各自处理？
- Prompt caching 怎么抽象？不同 provider 的 cache 机制差异太大
- 是否需要 model routing（根据任务复杂度选不同模型）？
- 本地模型（Ollama）的 tool calling 支持参差不齐，怎么降级？
