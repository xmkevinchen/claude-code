---
round: 1-2
date: 2026-04-04
score: converged
---

# Round 1-2 + Doodlestein

## Discussion

### Automation level
- /batch: 5-30 parallel agents, each in independent worktree, auto simplify→test→commit→PR. This is production code, not experiment.
- Coordinator mode: Research→Synthesis→Implementation→Verification 4-phase, "parallelism is your superpower"
- VERIFICATION_AGENT: code-complete adversarial verifier with nudge mechanism, but ant-internal A/B only
- Corrected estimate: 75-85% of execution-layer work automatable (up from 60-70%)
- Execution chain = external prod; verification chain = ant-internal blueprint. Direction confirmed, timeline extended.

### Developer impact
- Junior: 1-3 year entry-level pressure (all agree). Not linear disappearance — stepwise compression. "AI supervisor" is senior engineering with higher leverage, not a new role (Codex).
- Senior: value shifts from "write good code" to "define precise specs + judge AI output"
- Architect: short-term safe, long-term erosion as ULTRAPLAN/PlanAgent mature

### Key original insight: 分化轴 = 系统性判断力
- "会写代码 = 技术人才" 这个等式正在失效
- PM with strong systems judgment may use these tools MORE effectively than Junior Dev
- AI tools redefine "technical talent" — from coding ability to systems judgment

### Deception risk for non-technical users
1. /batch 92% success rate hides whether failed 2/24 are critical path
2. PARTIAL verdict misread as "mostly pass" by outsiders
3. Coordinator synthesis is opaque — users can't see assumptions

### BYOC signal (corrected)
- BYOC/Self-Hosted Runner = Anthropic's enterprise infra strategic intent (register/heartbeat API)
- Corrected from "paradigm shift" to "intent signal" — adoption speed determines impact timeline
- Architecture for "AI as infrastructure" is in place; organizational impact is projection, not code fact

## Outcome
- Score: converged
- Decision: 75-85% execution automation, verification chain is blueprint. 分化轴=系统性判断力 is the key original conclusion. BYOC is intent signal for AI-as-infrastructure.
