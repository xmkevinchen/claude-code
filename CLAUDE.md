# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is a source-only mirror of Claude Code (Anthropic's AI coding CLI), extracted from sourcemaps bundled in the npm package. There is **no package.json, no build config, no tests** — just the TypeScript source. You cannot build or run this project directly.

## Architecture

**Entry flow:** `src/entrypoints/cli.tsx` → `src/main.tsx` (Commander.js + React/Ink CLI)

**Core engine:**
- `QueryEngine.ts` — LLM query execution, streaming, context management
- `query.ts` — Query loop with tool execution orchestration
- `Tool.ts` + `tools.ts` — Base tool interface and registry (40+ tools in `src/tools/`)
- `commands.ts` + `src/commands/` — 100+ slash commands routed via Commander.js

**Terminal UI:** `src/ink/` is a **fully custom React terminal renderer** (not the Ink npm package). Includes ANSI parsing (`Ansi.tsx`), DOM/layout engine, focus management. `src/components/` builds on this.

**Key services (`src/services/`):**
- `api/claude.ts` — Claude API client with retry, token estimation, prompt caching, cost tracking
- `mcp/client.ts` — MCP server discovery, connection management, tool bridging
- `autoDream/` — Background memory consolidation ("dreaming")
- `compact/` — Auto-compaction when context length exceeds limits
- `oauth/` — Multi-provider auth (Claude AI, GitHub, Slack)
- `plugins/` — Plugin system for extensibility

**Multi-agent (`src/coordinator/`, `src/tools/AgentTool/`):**
- Swarm coordination via `coordinatorMode.ts`
- `AgentTool` spawns subagents, `SendMessageTool` for inter-agent messaging
- Shared task queue via `TaskCreateTool`/`TaskUpdateTool`

**IDE integration:** `src/bridge/bridgeMain.ts` — WebSocket bridge for VS Code, JetBrains, Cursor

**State:** React context via `src/state/AppState.tsx` + `AppStateStore.ts`, plus contexts in `src/context/` for notifications, stats, overlays, voice, modals.

**Permission system:** `src/utils/permissions/` + `useCanUseTool` hook. Modes: default (always ask), bypass, auto (rule-based). Rules in `~/.claude/permissions.json`.

**Feature flags:** `feature('KAIROS')`, `feature('COORDINATOR_MODE')`, `feature('DAEMON')` etc. — conditional compilation via Bun's bundler DCE.

## Notable Internal Systems

- **Buddy** (`src/buddy/`): Tamagotchi companion with deterministic gacha (Mulberry32 PRNG), 18 species, stats
- **Undercover** (`src/utils/undercover.ts`): Filters internal model codenames in public repos
- **KAIROS**: Proactive always-on assistant (feature-flagged)
- **ULTRAPLAN**: Offloads complex planning to remote Opus sessions
- **Hooks** (`src/hooks/`, 87 files): Permission checks, keybindings, background tasks, pre/post-query extensibility

## Navigating Large Files

Several files are extremely large (bundled/generated): `ink/ink.tsx` (251K lines), `cli/print.ts` (212K), `utils/ansiToPng.ts` (214K), `services/api/claude.ts` (125K), `bridge/bridgeMain.ts` (115K). Use targeted reads with offset/limit rather than reading these in full.
