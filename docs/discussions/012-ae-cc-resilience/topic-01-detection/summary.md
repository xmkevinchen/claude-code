---
id: "01"
title: "Detection — AE Knows When CC Changes"
status: pending
current_round: 0
created: 2026-04-05
decision: ""
rationale: ""
reversibility: ""
---

# Topic: Detection — AE Knows When CC Changes

## Current Status
No discussion yet. AE currently has zero detection mechanism for CC changes.

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
AE depends on CC tool names (Agent, SendMessage, TeamCreate, etc.), tool parameters (isolation, run_in_background, team_name), env vars (CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS), and behavioral assumptions (worktree paths, task-notification XML format). None of these are version-locked. When CC updates:
- Tool renames → AE prompt references stale names → model can't find tool → silent failure or workaround
- Parameter changes → AE passes invalid params → tool rejects or ignores → silent degradation
- Feature flag removal → AE pre-checks fail or pass incorrectly

Prior art: ToolSearch tool exists and can verify tool availability at runtime. ae:setup already does pre-checks.

## Constraints
- AE is prompt-only (SKILL.md) — cannot run TypeScript code at startup
- Detection must work within a skill execution context (tool calls available)
- Must not add significant latency to every skill invocation
- CC does not publish a public API version or changelog for internal tools
- ToolSearch can verify tool existence but not parameter schemas

## Key Questions
1. Where should detection run? Every skill invocation, ae:setup only, or a dedicated health-check skill?
2. What should be checked? Tool existence only (ToolSearch), or also parameter validation (try a no-op call)?
3. How to maintain the dependency list? Manual file, auto-generated from SKILL.md parsing, or both?
4. What's the notification mechanism when a problem is detected? Warning and continue, or block?
