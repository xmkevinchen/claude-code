---
round: 1
date: 2026-04-04
score: converged
---

# Round 1

## Discussion

**Gemini** outlined: bash coordinator + fan-out as simplest viable; checkpoint pattern for resumable pipelines; rate limits as #1 failure mode of parallel execution.

**Codex** recommended: Python orchestrator for non-trivial flows; SQLite WAL for task queue; lease-based claiming for dead worker recovery.

**Architect** recommended: tmux for session keepalive; durable cron for periodic triggers; hooks for observability only. External orchestrator + `claude -p` + git worktrees as the standard pattern.

**Code-researcher** confirmed: no session-level lock prevents concurrent `claude -p`; cron scheduler lock is the only contention point (one session owns it).

Team consensus on failure modes (ranked):
1. Rate limits (HIGH×HIGH) — mitigate with staggered launches
2. Context drift (HIGH×MEDIUM) — sequential for dependent tasks
3. Orphaned processes (MEDIUM×HIGH) — `timeout` wrapper + exit code checking
4. File contention (LOW×HIGH) — one worktree per worker
5. Session resume fragility (LOW×LOW) — design stateless, don't rely on resume

## Outcome
- Score: converged
- Decision: External orchestrator (bash for simple, Python for complex) + `claude -p` + git worktrees + file-based task queue (JSON/SQLite). tmux for session keepalive. Hooks for monitoring. Rate limit mitigation via staggered launches.
