---
id: "02"
title: "Integration Branch + Merge Flow Design"
status: converged
current_round: 1
created: 2026-04-05
decision: "Integration branch default ON (opt-out available). Auto-create batch/<slug> from main HEAD. Workers PR to it. Sequential merge without per-merge CI. Coordinator creates final PR to main."
rationale: "Protection of main is real value even without post-merge validation. Concurrent auto-merge produces race conditions. No branch protection needed on throwaway integration branch."
reversibility: "high"
reversibility_basis: "Feature is additive with opt-out flag — can disable without affecting other functionality"
---

# Topic: Integration Branch + Merge Flow Design

## Current Status
Converged — integration branch default ON, sequential merge, final PR to main.

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|
| 1 | converged | Consensus verified: advocate defended protection-of-main value, conceded per-merge CI and opt-in flag |
