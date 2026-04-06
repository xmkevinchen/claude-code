---
id: "010"
title: "/batch Skill Deep Analysis — Conclusion"
concluded: 2026-04-05
plan: ""
---

# /batch Skill Deep Analysis — Conclusion

## Decision Summary (Converged)

| # | Topic | Decision | Rationale | Reversibility |
|---|-------|----------|-----------|---------------|
| 1 | Wrapper Architecture | Build parallel `/ae:batch` as standalone SKILL.md. Minimize replicated surface (~20-25 lines of delta). Describe /batch's base phases by intent, not verbatim copy. Only replicate what AE modifies (worker instructions, integration branch setup). | `disableModelInvocation: true` blocks SkillTool invocation. No hook points between phases. All /batch primitives are standard AgentTool params. | high |
| 2 | Integration Branch | Default ON (opt-out available). Auto-create `batch/<slug>` from main HEAD. Workers use `gh pr create --base batch/<slug>`. User merges PRs manually using advisory sequence. Coordinator handles bookends only (create branch + create final PR to main). No branch protection on integration branch. | Protection of main is real value. Conflicts surface on throwaway branch. User-driven merge avoids coordinator bottleneck. Branch protection on integration branch is unnecessary (coordinator-owned scratch). | high |
| 3 | Dependency Analysis | v1: Advisory merge sequence in Phase 1 plan output. Coordinator suggests PR merge order based on file overlap. Non-blocking, non-automated. LSP-based analysis deferred to v1.1. | Zero tooling cost (few lines in existing plan output). Workers are permanently isolated — conflicts only materialize at merge time. | high |
| 4 | Scope Boundaries | **v1 MUST**: standalone `/ae:batch` + integration branch + advisory merge sequence + final PR to main. **Refinements**: (a) minimize prompt replication via intent-based description + delta-only, (b) version-pinning comment with upstream commit hash, (c) CI check warning in Phase 3, (d) pipeline.yml explicit warning (not silent, not hard-require). **DEFER**: post-merge validation (v2), LSP dep analysis (v1.1), coordinator-as-subagent (v2), coordinator-driven serial merge (v2). **NEVER**: auto-rollback, auto-retry. | Every MUST prevents a concrete first-run failure. Every DEFER requires real-world evidence or introduces new failure modes. Minimized replication surface reduces 6-month drift risk. | high |

## v1 Skill Architecture

```
/ae:batch <instruction>

Phase 0 — Pre-flight (AE-specific):
  ├─ Check git repo (same as /batch)
  ├─ gh auth status check (new — prevents 30 workers failing at PR step)
  ├─ Create integration branch: git checkout -b batch/<slug> && git push origin batch/<slug>
  ├─ pipeline.yml check: warn if absent (not block)
  └─ CI warning: note if CI may not trigger on batch/<slug>

Phase 1 — Research & Plan (intent-referenced from /batch):
  ├─ Enter plan mode
  ├─ Research subagents → decompose into 5-30 units
  ├─ [AE addition] Advisory merge sequence in plan output
  ├─ [AE addition] e2e recipe with AskUserQuestion fallback
  └─ Exit plan mode → user approval

Phase 2 — Spawn Workers (intent-referenced + AE delta):
  ├─ Spawn background agents with isolation: "worktree"
  └─ Worker instructions delta:
       - gh pr create --base batch/<slug> (instead of default)
       - Labels: ae-batch-<id>, ae-unit-<n>
       - Rest: standard worker pipeline (simplify → test → commit → push → PR)

Phase 3 — Track & Close (intent-referenced + AE delta):
  ├─ Track via status table (same as /batch)
  ├─ [AE addition] CI check warning if needed
  ├─ [AE addition] Final: gh pr create --base main --head batch/<slug>
  └─ [AE addition] Cleanup recipe: git push origin --delete batch/<slug>
```

## Doodlestein Review

| Agent | Challenge | Resolution |
|-------|-----------|------------|
| Strategic | Add `baseBranch` param to `getOrCreateWorktree` | Dismissed — AE cannot modify CC internals. Workers branching from main is acceptable (GitHub computes diff against target branch). |
| Adversarial | Integration branch creates merge bottleneck. Who merges? Ship v1 without it. | Resolved — coordinator can run `gh pr merge` (has Bash access). Phase 3 must explicitly include sequential merge instructions. Integration branch is net-positive: conflicts surface on throwaway branch, not main. Without it, half-merged main blocks the team. |
| Regret | T1 prompt replication is highest 6-month reversal risk. Build composition layer instead. | Resolved with 2 mitigation options. **Primary**: minimize replicated surface to ~25 lines of AE delta, describe base phases by intent, structurally isolate derived-from-CC vs AE-owned sections. **Alternative to explore**: three-step flow (`/ae:batch-setup` → user runs native `/batch` → `/ae:batch-finalize`) — zero replication, clean separation, but worse UX. Evaluate during implementation. |

## Spawned Discussions

None.

## Deferred Resolutions

| # | Item | Resolution | Detail |
|---|------|------------|--------|
| 1 | Post-merge validation | v2 | Per-worker tests + CI on integration branch sufficient for v1. Add coordinator validation step when evidence shows CI coverage is inadequate. |
| 2 | LSP-based dependency analysis | v1.1 | Requires language-specific tooling. Advisory merge sequence covers the guidance need for v1. Add when users encounter failures that advisory sequence doesn't prevent. |
| 3 | Coordinator-as-subagent | v2 | Inline coordinator is sufficient for 5-30 workers. Promote to subagent if context exhaustion is observed in practice. |
| 4 | Coordinator-driven serial merge | v2 | User-driven merge in v1. Add autonomous merge agent when the skill has enough usage to justify the complexity. |

## Team Composition

| Agent | Role | Backend | Joined |
|-------|------|---------|--------|
| team-lead | TL (moderator) | Claude Opus | Start |
| architect | System design | Claude | Start |
| code-researcher | CC internals evidence | Claude | Start |
| codex-proxy | Cross-family (orchestration) | Codex → Sonnet fallback | Start |
| gemini-proxy | Cross-family (UX/scope) | Gemini → Sonnet fallback | Start |
| doodlestein-strategic | Strategic improvement | Claude | Doodlestein |
| doodlestein-adversarial | Blind-spot detection | Claude | Doodlestein |
| doodlestein-regret | Regret prediction | Claude | Doodlestein |

## Process Metadata

- Discussion rounds: 2 (Round 1 independent research, Round 2 cross-discussion)
- Topics: 4 total (4 converged, 0 spawned, 0 deferred)
- Autonomous decisions: 4
- User escalations: 0
- Consensus verifications: 2 (Topic 2 integration branch, Topic 4 scope)
- Doodlestein challenges: 3 raised, 3 resolved, 0 reopened topics
- Deferred resolved in Sweep: 0 (no deferred items — all resolved)

## Next Steps

→ `/ae:plan` for `/ae:batch` v1 implementation based on converged decisions
→ Key implementation artifact: SKILL.md file using intent-referenced architecture with ~25-line delta
