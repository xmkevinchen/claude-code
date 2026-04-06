---
id: "02"
title: "Context Overlap Heuristic for Agent Reuse"
status: converged
current_round: 1
created: 2026-04-05
decision: "Add 4-row context overlap table to ae:work Execution Mode Selection. AE-specific cases only (cross-step reuse, error correction, verification = fresh). Not ae:agent-teams."
rationale: "Full CC table designed for transient workers; AE has persistent teammates. Only ae:work has the continue-vs-spawn decision point. 4 rows cover all AE-relevant cases."
reversibility: "high"
reversibility_basis: "Reference table in SKILL.md — easily updated or removed"
---

# Topic: Context Overlap Heuristic for Agent Reuse

## Current Status
Converged — 4-row table in ae:work.

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|
| 1-2 | converged | 4-0: ae:work scope, 4-row table, not ae:agent-teams |
