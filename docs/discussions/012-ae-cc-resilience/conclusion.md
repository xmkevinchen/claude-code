---
id: "012"
title: "AE-CC Resilience Strategy — Conclusion"
concluded: 2026-04-05
plan: ""
---

# AE-CC Resilience Strategy — Conclusion

## Executive Summary

AE 对 CC 的依赖面窄且稳定（工具名是常量，CC 维护向后兼容别名）。唯一确认的脆弱点是 `run_in_background` 参数可被条件性移除。应对策略：3 个针对性改动，不引入新 skill、不增加每次调用延迟、不膨胀 SKILL.md。

## Decision Summary (Converged)

| # | Topic | Decision | Rationale | Reversibility |
|---|-------|----------|-----------|---------------|
| 1 | Detection | Extend ae:agent-teams tier trigger: "env var missing OR tool not found via ToolSearch." Add run_in_background param check before team spawn. Document dependencies in CLAUDE.md. | ToolSearch can verify tool existence at runtime (zero cost). `run_in_background` conditional gating is the ONE confirmed fragility. Tool names are stable (constants pattern + backward compat aliases). | high |
| 2 | Graceful Degradation | Extend existing 3-tier model: "tool not found → same tier as env var missing." `run_in_background` missing → hard-block with `[BROKEN]` diagnostic. No per-skill degradation clauses. | Existing tier model is sound; just needs wider trigger. Per-skill "if available" clauses create complexity tax that degrades model compliance (Gemini's argument). Central protection in ae:agent-teams is cleaner. | high |
| 3 | Recovery Process | ae:setup adds tool verification step. CLAUDE.md block as human-readable dependency reference. `[BROKEN]` diagnostic points to recovery path. No dedicated health-check skill. | ae:setup already exists as maintenance entry point. No new skill needed — adding verification to existing flow. Manifest as plugin-level doc, not per-project file. | high |

## Executable Changes (可执行的应对策略)

### Change 1: ae:agent-teams SKILL.md — Extend Degradation Trigger

Location: Degradation section (after line 356)

Add to the tier table's trigger condition:

```markdown
### Extended Trigger Conditions

The degradation tiers apply when ANY of these conditions are true:
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env var is not set (existing)
- `TeamCreate` tool not found via ToolSearch (new)
- `Agent` tool schema missing `run_in_background` parameter (new — hard-block all tiers)

When `run_in_background` is unavailable, ALL team skills hard-block:
  [BROKEN] Background agent spawning disabled in this CC version.
  Agent Teams requires run_in_background support. Update Claude Code.
```

### Change 2: ae:setup SKILL.md — Add Tool Verification Step

Location: After Agent Teams enablement step

```markdown
### Tool Verification

After enabling Agent Teams, verify core tool availability:
1. ToolSearch("select:Agent,SendMessage,TeamCreate,TaskCreate")
2. Check all 4 tools returned
3. If any missing → warn: "⚠️ [tool] not found. AE may not work correctly."
4. ToolSearch("select:Agent") → verify schema includes `run_in_background`
5. If missing → warn: "⚠️ Background tasks disabled. Agent Teams will not work."

Write verification result to setup output.
```

### Change 3: CLAUDE.md — Add AE Dependencies Block

Location: Project CLAUDE.md or `~/.claude/CLAUDE.md`

```markdown
## AE Plugin Dependencies
- Required tools: Agent, SendMessage, TeamCreate, TaskCreate/Update/Get/List/Stop
- Required params: Agent.run_in_background, Agent.team_name, Agent.name
- Required env: CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS
- Known conditional: run_in_background gated on isBackgroundTasksDisabled (low risk — Agent Teams fundamentally requires background spawning; this gate is for CI/headless environments, not normal usage)
- If tools missing: AE degrades per ae:agent-teams tier table
- If [BROKEN] appears: check this list, verify CC version, report to AE maintainer
```

### Change 4: Convention — `[BROKEN]` Diagnostic Prefix

When AE encounters unexpected CC behavior that doesn't match known degradation paths:

```
[BROKEN] <specific description of what failed>
Check AE dependencies in CLAUDE.md. Verify CC version. 
Report to AE maintainer with this message.
```

This is a convention, not code — any AE skill that detects unexpected behavior uses this prefix. It's the "catch-all" for behavioral drift that can't be machine-detected.

## What We Deliberately Did NOT Add

| Rejected | Why |
|----------|-----|
| Dedicated ae:health-check skill | ae:setup already covers verification; new skill = maintenance burden |
| Per-skill "if available" clauses | Complexity tax degrades model compliance (Gemini argument) |
| Project-local dependencies.md | Goes stale per-project; CLAUDE.md block is sufficient |
| 4-layer resilience stack | Over-engineering for a prompt-only community plugin |
| Preflight check every session | Adds latency to every invocation for a low-frequency risk |
| Parameter scanning framework | Only ONE param (run_in_background) has confirmed fragility |

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Tool name renamed | Very low (constants pattern + backward compat aliases) | High (all AE team skills break) | ae:agent-teams tier trigger + ToolSearch |
| `run_in_background` disabled | Very low (Agent Teams fundamentally requires it; gate is for CI/headless, not normal usage) | High (team spawns serialize → TL hangs) | [BROKEN] convention — not worth a dedicated check |
| Parameter renamed | Low (inline Zod, no constants) | Medium (specific tool calls fail) | [BROKEN] convention → manual recovery |
| Behavioral drift (XML format, paths) | Low | Medium-high (silent degradation) | [BROKEN] convention → CLAUDE.md checklist |
| Feature flag renamed | Very low | Medium (pre-checks pass incorrectly) | CLAUDE.md documents known flags |
| env var removed entirely | Very low (Agent Teams is a core feature) | High | ae:agent-teams tier trigger |

## Team Composition

| Agent | Role | Backend | Joined |
|-------|------|---------|--------|
| team-lead | TL (moderator) | Claude Opus | Start |
| architect | Resilience architecture | Claude | Start |
| code-researcher | CC + AE code evidence | Claude | Start |
| codex-proxy | Plugin resilience patterns | Codex → Sonnet fallback | Start |
| gemini-proxy | Pragmatism / complexity budget | Gemini → Sonnet fallback | Start |

## Process Metadata

- Discussion rounds: 2
- Topics: 3 (3 converged)
- Autonomous decisions: 3
- User escalations: 0
- Doodlestein: skipped (pragmatic fixes, not architectural choices)

## Next Steps

→ 这 4 个改动可以直接实现到 AE SKILL.md 文件中
→ Change 1 (tier extension) 和 Change 2 (setup verification) 应一起实现
→ Change 3 (CLAUDE.md block) 立即可做
→ Change 4 ([BROKEN] convention) 是约定，不需要代码改动
