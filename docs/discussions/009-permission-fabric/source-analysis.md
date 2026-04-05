---
id: "009"
title: "Permission Fabric — Source Analysis"
type: analysis
created: 2026-04-05
origin: "Claude Code source exploration (007) + permission system deep dive"
tags: [product, permissions, security, ai-safety]
---

# Permission Fabric — Source Analysis

## What Claude Code Actually Built

Claude Code's permission system is 23+ files under `src/utils/permissions/`, plus hooks, React integration, and team coordination handlers. It's the most sophisticated AI agent permission system in any shipping product, and nobody outside Anthropic has built anything comparable.

### The Core Problem It Solves

AI agents need to take actions in the real world — run shell commands, edit files, call APIs, spawn sub-agents. Some actions are safe (read a file), some are dangerous (delete a directory), and some depend on context (running `npm install` in a test project vs production). The permission system decides, for each action, whether to allow silently, deny silently, or ask the human.

This is the same problem every company building AI agents will face. Right now everyone rolls their own ad-hoc solution.

---

## Architecture: Five Layers

### Layer 1: Rule Engine

**What it does**: Pattern-matching rules that map tool invocations to allow/deny/ask decisions.

**How rules work** (`permissionRuleParser.ts:43-152`):

```
Bash                    → any bash command
Bash(ls:*)              → any command starting with "ls"
Bash(npm run build)     → exact command match
FileWrite(/src:*)       → writes under /src
Agent                   → any agent spawn
```

Three behaviors: `allow` (silent pass), `deny` (silent block), `ask` (prompt human).

**Rule sources, in priority order** (`permissions.ts:213-231`):

| Source | Where | Shared? | Example Use |
|--------|-------|---------|-------------|
| `policySettings` | Enterprise MDM push | Yes (org-wide) | "Nobody runs `rm -rf` ever" |
| `projectSettings` | `.claude/settings.json` (git-tracked) | Yes (team) | "This project allows `npm test`" |
| `localSettings` | `.claude/settings.local.json` (gitignored) | No (personal) | "I trust `docker` commands" |
| `userSettings` | `~/.claude/settings.json` | No (personal) | "Always allow `git status`" |
| `flagSettings` | `--settings file.json` CLI arg | No (session) | One-off config |
| `session` | In-memory | No (session) | User clicked "allow for this session" |

**Evaluation order**: Deny rules checked first → Ask rules → Allow rules → Default behavior. Within each tier, source priority applies (policy > project > local > user).

This means an org policy deny rule can't be overridden by a personal allow rule. Security cascades downward.

### Layer 2: Dangerous Pattern Detection

**What it does**: Automatically identifies high-risk commands that shouldn't be auto-allowed, even if the user has broad rules.

**Patterns stripped when entering auto mode** (`dangerousPatterns.ts:18-80`):

- **Interpreter invocations**: `python`, `node`, `ruby`, `perl`, `php` — because they can execute arbitrary code
- **Package runners**: `npx`, `bunx`, `npm run` — because they run untrusted code
- **Shells**: `bash`, `sh`, `zsh`, `ssh` — because they're escape hatches
- **Evaluators (PowerShell)**: `Invoke-Expression`, `Invoke-Command`, `Start-Process` — .NET escape hatches
- **Agent spawns**: Any `Agent` tool allow rule — because sub-agents bypass parent's classifier

**Why this matters for a product**: This is a curated, battle-tested catalog of "things AI agents shouldn't auto-do." It took real incidents to build. Anyone building AI agent permissions from scratch will eventually rediscover these same patterns, probably the hard way.

### Layer 3: LLM Classifier (Auto Mode)

**What it does**: Uses a side-query to a Claude model to decide if an action is safe, given the full conversation context. This is the "brain" of the permission system.

**Currently ant-internal only** — gated by `feature('TRANSCRIPT_CLASSIFIER')`.

**How it works** (`yoloClassifier.ts`):

1. User enters auto mode (or starts with `--auto-mode`)
2. For each tool invocation that isn't in the safe-allowlist or covered by explicit rules:
   - Build a transcript of the conversation (user text + tool_use blocks, NO assistant text to prevent injection)
   - Send to classifier with system prompt + user's custom allow/deny rules + environment context
   - Classifier returns `{ shouldBlock: boolean, reason: string, thinking: string }`
3. If blocked → deny with reason. If allowed → execute silently.

**Key design decisions**:
- **Transcript sanitization** (`yoloClassifier.ts:287-360`): Only user messages and tool_use blocks go to classifier. Assistant text is excluded because the AI could craft text that manipulates its own classifier. This is a prompt injection defense.
- **Denial limits** (`denialTracking.ts:1-46`): Max 3 consecutive denials or 20 total per session → falls back to interactive prompting. Prevents classifier denial loops from permanently blocking the user.
- **Safe allowlist** (`classifierDecision.ts:56-94`): Read-only tools (Grep, Glob, Read), task management, and UI tools skip the classifier entirely. No point burning tokens on clearly safe operations.
- **Custom rules as classifier input**: User's allow/deny rules are injected into the classifier prompt as guidance. "The user trusts `npm test`" biases the classifier toward allowing npm test. This is rule + AI hybrid, not pure rule or pure AI.

**What makes this special**: Most AI permission systems are either pure rule-based (brittle, can't handle novel situations) or pure AI-based (unpredictable, hard to audit). Claude Code's approach is hybrid — rules for known patterns, AI for novel situations, with the AI constrained by rules and safety limits. This hybrid architecture is the core IP.

### Layer 4: Team Permission Coordination

**What it does**: When multiple agents work together (coordinator + workers), permissions are synchronized across the team.

**Coordinator mode** (`coordinatorHandler.ts:26-62`):
- Main agent has full permission authority
- Runs hooks → tries classifier → falls back to interactive dialog

**Worker mode** (`swarmWorkerHandler.ts:40-100+`):
- Workers CAN'T prompt the user (non-interactive context)
- Try classifier first (auto-approve safe commands)
- Forward uncertain decisions to coordinator via mailbox (`permissionSync.ts`)
- Wait for coordinator's answer
- Coordinator decides on behalf of worker

**This is the distributed policy enforcement pattern**. In a multi-agent system, you can't have every agent asking the user separately — that would be unbearable. But you can't give every agent bypass permissions either. The coordinator acts as a permission proxy, making decisions on behalf of workers using the same rules and classifier.

### Layer 5: Enterprise Policy Override

**What it does**: Organizations can push mandatory permission rules via MDM (Mobile Device Management) that override all user/project settings.

**`allowManagedPermissionRulesOnly`** (`permissionsLoader.ts:31`):
- When enabled, only policy rules are respected
- User and project settings rules are ignored
- "Always allow" options are hidden from the UI
- Users can't accidentally weaken security

**GrowthBook killswitches**:
- `tengu_bypass_permissions_config.allowBypassMode` — can disable bypass mode globally
- `tengu_auto_mode_config.enabled` — can disable auto mode globally
- Both can be flipped remotely without code deploy

This is the "compliance" layer. An enterprise using AI agents needs to ensure that no individual developer can grant their agent more access than policy allows.

---

## What Doesn't Exist Anywhere Else

No other AI agent framework has all five of these:

| Capability | Claude Code | LangChain | AutoGen | CrewAI | Cursor |
|-----------|-------------|-----------|---------|--------|--------|
| Rule-based allow/deny/ask | ✅ Full pattern matching | ❌ | ❌ | ❌ | Partial |
| Dangerous pattern catalog | ✅ Shell + PS + agent | ❌ | ❌ | ❌ | ❌ |
| AI-based classifier | ✅ (ant-only) | ❌ | ❌ | ❌ | ❌ |
| Multi-agent permission sync | ✅ Coordinator/worker | ❌ | ❌ | ❌ | ❌ |
| Enterprise policy override | ✅ MDM + killswitch | ❌ | ❌ | ❌ | ❌ |
| Shadow rule detection | ✅ | ❌ | ❌ | ❌ | ❌ |
| Denial tracking / fallback | ✅ | ❌ | ❌ | ❌ | ❌ |

Most agent frameworks have a "yes/no" human-in-the-loop check, and that's it. No rule hierarchy, no pattern detection, no AI classifier, no multi-agent coordination, no enterprise policy.

---

## What This Means as a Product

### The problem is universal

Every company building AI agents hits the same wall: "How do I let the AI do things without letting it do everything?" Ad-hoc solutions multiply: hardcoded allowlists, simple yes/no prompts, "just trust the AI" (until the incident).

### The market doesn't have a standard

OPA (Open Policy Agent) standardized policy enforcement for Kubernetes. There's no equivalent for AI agents. Every team reinvents:
- What's a safe action? (Claude Code's dangerous pattern catalog)
- How to decide edge cases? (Claude Code's hybrid rule+AI classifier)
- How to coordinate across multiple agents? (Claude Code's coordinator/worker sync)
- How to enforce org-wide policy? (Claude Code's MDM override)

### Claude Code proved the architecture works

This isn't a theoretical design. It's been running in production, handling real tool invocations from real users. The denial tracking, shadow rule detection, and classifier fallback patterns exist because they were needed — they're solutions to real problems that emerged in production.

---

## Product Concept: Permission Fabric

### One-liner

Policy engine for AI agent actions — rule-based, AI-augmented, multi-agent aware, enterprise-ready.

### What it does

A middleware layer that sits between AI agents and the tools/APIs they invoke. For each action, it decides: allow, deny, or escalate to human.

### How it's different from Claude Code's implementation

Claude Code's permission system is tightly coupled to Claude Code's internals — React hooks, AppState, MCP tool definitions, specific file paths. Permission Fabric would be:

- **Agent-agnostic**: Works with Claude Code, Cursor, AutoGen, LangChain, custom agents — anything that invokes tools
- **Deployment-agnostic**: Embedded library, sidecar service, or MCP server
- **Model-agnostic**: AI classifier can use any LLM, not just Claude

### Architecture

```
AI Agent (any framework)
    │
    │ "I want to run: npm install express"
    ↓
Permission Fabric
    ├── Layer 1: Rule engine (allow/deny/ask patterns)
    ├── Layer 2: Dangerous pattern detector (known-risky commands)
    ├── Layer 3: AI classifier (novel situations, context-aware)
    ├── Layer 4: Multi-agent coordinator (delegate decisions)
    └── Layer 5: Org policy override (enterprise MDM)
    │
    ↓
Decision: allow / deny / escalate
    │
    ↓
Tool execution (or human prompt)
```

### Connection to Second Brain

Permission Fabric controls what agents can **do**. Second Brain controls what agents can **know**. At the org level, these converge:

| Permission Fabric | Second Brain |
|-------------------|-------------|
| `policySettings` (org deny rules) | Org knowledge access policy |
| `allowManagedPermissionRulesOnly` | "Only approved knowledge enters org memory" |
| Rule hierarchy (policy > project > user) | Memory hierarchy (org > team > personal) |
| AI classifier for novel situations | Dreaming classifier for novel knowledge |
| Coordinator/worker permission sync | Team memory promotion/access sync |

A unified "AI governance layer" could combine both: what agents do + what agents know, controlled by the same policy hierarchy.

---

## Extractability from Claude Code

### What's directly portable

| Component | Coupling to Claude Code | Extraction Cost |
|-----------|------------------------|-----------------|
| Rule format + parser (`permissionRuleParser.ts`) | Low — pure string parsing | ~1 day |
| Dangerous pattern catalog (`dangerousPatterns.ts`) | Zero — plain data | Hours |
| Rule evaluation logic (`permissions.ts:213-231`) | Medium — needs decoupling from settings loader | ~2 days |
| Shadow rule detection (`shadowedRuleDetection.ts`) | Low — pure logic | ~1 day |
| Shell rule matching (`shellRuleMatching.ts`) | Low — pure matching | ~1 day |
| Denial tracking (`denialTracking.ts`) | Zero — pure state machine | Hours |

### What needs rewriting

| Component | Why | Effort |
|-----------|-----|--------|
| AI classifier (`yoloClassifier.ts`) | Deeply coupled to Claude Code's API client, model resolution, prompt caching | 1-2 weeks |
| Team coordination (`swarmWorkerHandler.ts`, `permissionSync.ts`) | Coupled to mailbox system, AppState, React hooks | 1-2 weeks |
| Enterprise policy (`permissionsLoader.ts`) | Coupled to GrowthBook, MDM integration | 1 week |
| React integration (`useCanUseTool.tsx`, `PermissionContext.ts`) | Coupled to UI layer — not needed for a library | N/A (skip) |

### Total extraction estimate

Core library (layers 1-2, no AI classifier, no team coordination): **~1 week**
Full system (all 5 layers): **~4-5 weeks**

---

## Key Design Decisions Worth Preserving

1. **Deny-first evaluation**: Deny rules always win. No lower-priority source can override a higher-priority deny. This is a security invariant.

2. **Transcript sanitization for classifier**: Never send assistant text to the classifier. The AI's own output could manipulate its own permission check. Only send user messages + tool invocations.

3. **Dangerous patterns stripped on mode transition**: When entering auto mode, broad rules (wildcard Bash, Agent spawns) are removed. The classifier should evaluate each action on its merits, not inherit blanket permissions.

4. **Denial limits with fallback**: After N denials, stop blocking and ask the human. Prevents the AI classifier from permanently deadlocking the user. Safety valve.

5. **Shadow rule detection**: Proactively warn when a rule is unreachable. Better to surface config errors at setup time than fail silently at action time.

6. **Rule + AI hybrid**: Rules for known patterns (fast, auditable, predictable). AI for novel situations (flexible, context-aware). Neither alone is sufficient.

---

## Open Questions

1. **Delivery format**: Embedded Rust library? MCP server (like Second Brain)? Sidecar? OPA-style declarative policy language? The answer depends on target customer — individual developer (library) vs enterprise (sidecar + policy language).

2. **Relationship to Second Brain**: Are these two products, or one product with two surfaces? The policy hierarchy is the same. The enforcement points are different (action control vs knowledge access). Could be one "AI Governance" layer.

3. **Business model**: Open-source core (rule engine + patterns) + commercial enterprise layer (AI classifier + team coordination + MDM)? This mirrors OPA's model (open core, commercial Styra).

4. **Classifier model dependency**: Claude Code uses Claude as the classifier. A product needs to work with any LLM. The classifier prompt and tool schema are model-agnostic, but quality varies across models. Need testing with GPT-4, Gemini, open-source models.

5. **Compliance standards**: Enterprise customers will ask about SOC2, GDPR, ISO 27001 implications. The permission decision audit trail needs to be compliance-ready from Day 1. Claude Code logs decisions but not in a compliance-friendly format.

6. **The "dangerous pattern" catalog**: This is a living document. New risks emerge as AI capabilities evolve. Who maintains it? Community-driven (like OWASP)? Vendor-curated? Both?
