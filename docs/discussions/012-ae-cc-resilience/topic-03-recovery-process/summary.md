---
id: "03"
title: "Recovery Process — What To Do When Broken"
status: pending
current_round: 0
created: 2026-04-05
decision: ""
rationale: ""
reversibility: ""
---

# Topic: Recovery Process — What To Do When Broken

## Current Status
No discussion yet. AE has no documented recovery process for CC-breaking changes.

## Round History
| Round | Score | Key Outcome |
|-------|-------|-------------|

## Context
When CC updates break AE, someone needs to:
1. Identify what changed in CC
2. Assess impact on AE skills
3. Update AE SKILL.md files
4. Test that AE works again

This is currently entirely manual and undocumented. The CC source mirror (this repo) helps for deep analysis but is itself a snapshot that can be stale.

Discussion 010 (batch analysis) introduced "version-pinning comments" — SKILL.md files note which CC version they were verified against. Discussion 011 introduced "dependencies.md" as a manifest. These are starting points but not a process.

## Constraints
- AE is community-maintained — recovery process must be simple enough for one person
- CC updates are opaque — no changelog for internal tool changes
- Recovery must not require deep CC source knowledge for every fix
- The CC source mirror may be stale — AE must work from observable behavior, not just source

## Key Questions
1. What's the minimal recovery checklist when CC updates?
2. Should AE have a `dependencies.md` manifest that lists all CC dependencies with verification commands?
3. Can recovery be partially automated? (e.g., ae:health-check skill that validates all dependencies)
4. What's the trigger for recovery — user reports breakage, or proactive post-update check?
