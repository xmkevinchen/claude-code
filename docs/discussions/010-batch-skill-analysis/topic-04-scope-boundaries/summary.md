---
id: "04"
title: "AE Batch — Scope Boundaries & What NOT to Build"
status: converged
current_round: 1
created: 2026-04-05
decision: "v1: standalone /ae:batch + integration branch + advisory merge sequence + final PR. Refinements: version-pinning comment, CI check warning, pipeline.yml explicit warning (not silent, not hard-require). Defer: post-merge validation (v2), LSP analysis (v1.1), subagent coordinator (v2). Never: auto-rollback, auto-retry."
rationale: "Every MUST prevents a concrete first-run failure. Every DEFER requires real-world evidence or introduces new failure modes. Critic refinements (maintenance comment, CI warning, explicit pipeline.yml warning) address HOW without changing WHAT."
reversibility: "high"
reversibility_basis: "Scope is additive — easy to expand or contract; single SKILL.md with no external dependencies"
---

# Topic: AE Batch — Scope Boundaries & What NOT to Build

## Current Status
Converged — v1 scope defined with 4 refinements from consensus verification.

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|
| 1 | converged | Consensus verified: advocate defended scope, conceded CI warning, explicit pipeline.yml warning, and version-pinning comment. Defended single SKILL.md (no Phase 3 split). |
