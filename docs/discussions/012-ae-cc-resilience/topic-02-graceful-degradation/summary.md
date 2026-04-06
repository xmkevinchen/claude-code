---
id: "02"
title: "Graceful Degradation — AE Doesn't Crash"
status: pending
current_round: 0
created: 2026-04-05
decision: ""
rationale: ""
reversibility: ""
---

# Topic: Graceful Degradation — AE Doesn't Crash

## Current Status
No discussion yet. AE has partial degradation (Agent Teams auto-fallback) but no general degradation framework.

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
AE agent-teams protocol (SKILL.md:349-356) already has a 3-tier degradation model for Agent Teams unavailability:
- hard-block: ae:discuss, ae:review — refuse to execute
- auto-fallback: ae:analyze, ae:plan — TL executes solo, lower confidence
- no-pre-check: ae:code-review — no dependency

But this only covers one failure mode (Agent Teams missing). It doesn't cover:
- Individual tool removal (SendMessage gone but Agent still works)
- Tool parameter changes (isolation:"worktree" no longer valid)
- Behavioral changes (task-notification format different)
- Feature flag changes (env var name changed)

The worst case is silent degradation: AE runs, produces output, but the output is wrong because assumptions are broken. This is more dangerous than a crash.

## Constraints
- AE SKILL.md prompts are instructions to an LLM — they can use "if X available then Y else Z" patterns
- Adding "if not available" clauses to every tool reference bloats the prompt
- Must balance resilience with prompt complexity (model compliance degrades with longer prompts)
- Some degradations are recoverable (tool renamed → use new name) vs others are not (tool removed entirely)

## Key Questions
1. Should degradation be per-tool (each tool reference has a fallback) or per-skill (each skill has a degradation mode)?
2. How to handle the "silent degradation" problem — AE runs but produces wrong output?
3. What's the minimum viable degradation framework that doesn't bloat SKILL.md?
4. Should AE's prompt language shift from "Call TeamCreate" to "If TeamCreate is available, call it; otherwise..."?
