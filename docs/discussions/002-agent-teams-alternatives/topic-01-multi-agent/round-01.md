---
round: 1
date: 2026-04-04
score: converged
---

# Round 1-2

## Discussion

**Architect** proposed three architectures:
- A: File-based IPC (mailbox emulation)
- B: tmux + `claude -p` external orchestrator  
- C: Hooks as event bus + `claude server`

Plus identified that AgentTool `run_in_background` + `isolation: worktree` is the zero-infra path for simple fan-out.

**Code-researcher** verified from source:
- AgentTool + run_in_background + worktree ALL work without Agent Teams (no feature gate)
- Multiple `claude -p` instances can run concurrently (no session-level lock)
- FileChanged hook can detect external file writes
- `claude server` gated by `DIRECT_CONNECT` — may not be available

**Codex** recommended: SQLite WAL as task queue, file-based IPC as default, Python for non-trivial orchestration.

**Gemini** recommended: bash coordinator + `claude -p` fan-out as simplest viable, git branches as state machine, rate limits as #1 failure mode.

**Architect Round 2** clarified: background agents are parent↔child only (no peer-to-peer), file mailbox needs lockfile discipline, hooks are for observability not coordination. Added: inbox JSON rewrite cost scales with message volume — use JSONL or SQLite instead.

## Outcome
- Score: converged
- Decision: Three-tier approach — (a) AgentTool background+worktree for fan-out, (b) file-based IPC for peer-to-peer, (c) external orchestrator for complex pipelines. Hooks reserved for observability.
