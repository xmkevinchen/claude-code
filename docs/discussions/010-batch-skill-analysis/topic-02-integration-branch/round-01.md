---
round: 1
date: 2026-04-05
score: converged
---

# Round 1-2 + Consensus Verification

## Discussion

**Round 1**: All 4 agents agreed integration branch is v1 must-have. Worktree base branch constraint confirmed (worktree.ts:284-302 — always branches from default branch, no override).

**Round 2 convergence**: Workers branch from main, PR to `batch/<slug>`. GitHub computes diff correctly. No rebase needed. Sequential merge (not auto-merge). Coordinator creates final PR to main.

**Consensus verification** (Architect advocate vs Codex critic):

Critic's strongest attacks:
1. Integration branch is conflict redirect, not prevention
2. Workers never test integrated state — "theater" without post-merge validation
3. Sequential merge with per-merge CI = 200 min for 20 workers
4. Branch protection unsolved
5. Alternative: default to main, opt-in `--integration-branch`

Advocate concessions:
- Conceded #3: revised to merge WITHOUT per-merge CI wait
- Partially conceded #5: opt-in flag valid, but default should be ON (safety-asymmetric)

Advocate defenses:
- #2: Protection of main is real value even without post-merge validation — broken integration branch ≠ broken main
- #1: Isolation to throwaway branch means failures are contained and recoverable
- #4: No protection on `batch/<slug>` is intentional — it's a coordinator-owned scratch branch

## Outcome
- Score: converged
- Decision: Integration branch default ON, opt-out available. Auto-create `batch/<slug>` from main HEAD. Workers use `gh pr create --base batch/<slug>`. Sequential merge without per-merge CI. Coordinator creates final PR to main. No branch protection on integration branch (throwaway by design). Branch cleaned up after final PR merge.
- Reversibility: high — feature is additive, opt-out available
