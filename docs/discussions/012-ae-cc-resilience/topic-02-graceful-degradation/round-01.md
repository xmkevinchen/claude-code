---
round: 1
date: 2026-04-05
score: converged
---

# Round 1-2

## Discussion
AE's 3-tier model (hard-block/auto-fallback/no-pre-check) is sound but only triggers on env var missing. Extension: same tiers apply when ToolSearch reports tool absent. `run_in_background` missing = hard-block with diagnostic (not silent degradation). [BROKEN] convention for undetectable behavioral drift.

Gemini's complexity-tax concern is valid: per-skill "if available" clauses are worse than the failure they prevent. Protection lives in ae:agent-teams (central) not per-skill.

## Outcome
- Score: converged
- Decision: Extend tier table: "tool not found → apply same tier as env var missing." run_in_background missing → hard-block with `[BROKEN]` diagnostic. No per-skill degradation clauses.
