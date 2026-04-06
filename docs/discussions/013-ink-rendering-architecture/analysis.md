---
id: "013"
title: "Analysis: Ink Terminal Rendering Architecture"
type: analysis
created: 2026-04-05
tags: [ink, renderer, terminal-ui, react, architecture, performance]
---

# Analysis: Ink Terminal Rendering Architecture

## Question

How does Claude Code's custom React terminal renderer (`src/ink/`) work end-to-end, and what are the architectural trade-offs vs. using npm Ink or other terminal UI frameworks?

## Findings

### Prior Art from Project Knowledge Base

Prior context: unavailable (no relevant results from Second Brain).

### Relevant Code

**Core renderer pipeline** (~80 files in `src/ink/`):

| File | Role |
|------|------|
| `src/ink/reconciler.ts` | Custom React reconciler (react-reconciler host config) |
| `src/ink/dom.ts` | DOM tree — `DOMElement`/`TextNode` with dirty tracking |
| `src/ink/layout/yoga.ts` | Yoga layout adapter (flexbox) |
| `src/native-ts/yoga-layout/index.ts` | TypeScript reimplementation of Yoga (~2,578 lines) |
| `src/ink/ink.tsx` | Main orchestrator — double-buffer, throttled render loop |
| `src/ink/renderer.ts` | Yoga → Output → Screen conversion |
| `src/ink/render-node-to-output.ts` | Tree walk, blit optimization, scroll clipping |
| `src/ink/output.ts` | Operation collector (write/blit/clip/clear/shift) |
| `src/ink/screen.ts` | Cell-level buffer with CharPool/StylePool/HyperlinkPool interning |
| `src/ink/log-update.ts` | Cell-by-cell diff, DECSTBM hardware scroll |
| `src/ink/optimizer.ts` | Patch optimizer (merge cursor moves, collapse styles) |
| `src/ink/terminal.ts` | Terminal write with synchronized output (DEC 2026) |
| `src/ink/Ansi.tsx` | ANSI string → React component spans |
| `src/ink/termio/` | Full ANSI parser/tokenizer (CSI, DEC, OSC, SGR, ESC) |
| `src/ink/focus.ts` | Focus manager with tab navigation, stack, click focus |
| `src/ink/events/` | Capture/bubble event system (mirrors browser model) |
| `src/ink/selection.ts` | Text selection state machine |
| `src/ink/hit-test.ts` | Mouse click/hover hit testing |
| `src/ink/bidi.ts` | Bidirectional text reordering |

**Entry point**: React commit → `resetAfterCommit` → Yoga layout → `scheduleRender` (throttled 16ms) → renderer → Output → Screen → LogUpdate (diff) → optimizer → `writeDiffToTerminal`.

### Architecture & Patterns

**1. Reconciler**: Uses `react-reconciler` with ConcurrentRoot (live UI) and LegacyRoot (off-screen search renders — workaround for cross-root scheduler leak). Host element types: `ink-root`, `ink-box`, `ink-text`, `ink-virtual-text`, `ink-link`, `ink-progress`, `ink-raw-ansi`. Implements React 19 host config.

**2. Layout**: Yoga flexbox via a TypeScript port (`native-ts/yoga-layout/`) behind an adapter interface (`layout/node.ts`). Not all nodes get yoga nodes — `ink-virtual-text` and `ink-link` are layout-free. Text nodes use measure functions (`wrapText`/`measureText`); `ink-raw-ansi` uses pre-baked dimensions.

**3. Cell-level rendering**: The `Screen` is a flat cell array with integer-interned values:
- `CharPool` — grapheme cluster → integer ID (O(1) comparison, zero string allocation in diff)
- `StylePool` — caches SGR transition strings by `(fromId, toId)` pairs
- `HyperlinkPool` — OSC 8 dedup, resets every 5 minutes to bound growth

**4. Diff pipeline**: `LogUpdate.diffEach` compares cells by integer ID. Emits structured `Diff` (patches: stdout, cursorMove, cursorTo, style, clear, hyperlink, hideCursor/showCursor). Optimizer merges/collapses patches in a single pass.

**5. Double-buffering**: `frontFrame`/`backFrame` swap prevents tearing. `prevFrameContaminated` flag tracks when selection overlay invalidates the previous frame (disables blit optimization for that frame).

**6. Performance optimizations**:
- 60fps throttle (`FRAME_INTERVAL_MS = 16`) with `leading: true, trailing: true`
- Blit optimization: unchanged subtrees copy from `prevScreen` instead of re-rendering
- DECSTBM hardware scroll: `SU`/`SD` sequences when terminal supports DEC 2026 synchronized output
- Synchronized output (`\x1b[?2026h/l`) for atomic frame writes
- `ink-raw-ansi` bypasses all text measurement for pre-formatted content

**7. Event system**: Full capture/bubble model mirroring the browser. `Dispatcher` with discrete (keyboard/click/focus) and continuous (resize/scroll/mousemove) priority mapping aligned with React scheduling priorities.

**8. Advanced features beyond npm Ink**: Mouse tracking, text selection with clipboard (OSC 52), scrollable boxes with virtual/sticky scroll, OSC 8 hyperlinks, alternate screen mode, bidirectional text, terminal focus tracking, Kitty keyboard protocol.

### Industry Practice Comparison

**vs. npm Ink (the obvious alternative)**:

| Aspect | npm Ink | Claude Code custom |
|--------|---------|-------------------|
| Rendering granularity | Line-level string diff | Cell-level integer diff |
| Layout engine | Yoga (official WASM) | Yoga (TypeScript port) |
| Mouse/selection | Not supported | Full support |
| Scrolling | Not supported | Virtual + sticky scroll + DECSTBM |
| Hyperlinks | Not supported | OSC 8 |
| React version | Not React 19 | React 19 (with caveats) |
| Maintenance | Community | Internal team |

**vs. other frameworks**: Architecture is closer to **Ratatui** (Rust) than Ink — cell buffer, double-buffering, structured patch/diff. This is the gold standard approach for high-performance terminal UIs.

**Competitive landscape**: Only Claude Code and OpenCode have built fully custom renderers among AI CLI tools. Gemini CLI uses stock Ink. GitHub CLI uses Go + Charm ecosystem. Most CLI tools delegate rendering to frameworks.

### Challenges & Disagreements

**Challenge 1: Fork, not ground-up build**
Code contains references to `vadimdemedes/ink` issues, `TODO(vadimdemedes):` comments, and upstream API patterns. This is an internalized fork that was heavily extended, not a from-scratch implementation. Maintenance implications: diverged from upstream with no path to pull fixes.

**Challenge 2: Two forked upstreams**
Both the renderer (from Ink) and the layout engine (TypeScript Yoga reimplementation replacing official WASM package) are forks. The Yoga port omits: aspect-ratio, content-box sizing, RTL direction. This is a double maintenance burden.

**Challenge 3: React 19 integration is fragile**
5+ call sites use `@ts-expect-error` to access undocumented `react-reconciler` internals (`updateContainerSync`, `flushSyncWork`, `flushSyncFromReconciler`). Also depends on internal Fiber shape for debug instrumentation. React version upgrades are high-risk.

**Challenge 4: Performance optimizations have documented failure modes**
- Blit optimization is invalidated whenever text selection is active (`prevFrameContaminated`)
- DECSTBM hardware scroll is disabled in tmux (no DEC 2026 support) — falls back to full-row rewrite
- `fullResetSequence_CAUSES_FLICKER` (literally named) fires on terminal resize and scrollback events
- Soft-wrapping is explicitly incomplete (TODO in `screen.ts`)

**Challenge 5: ROI question**
No telemetry tracks what % of users exercise advanced features (fullscreen, alt-screen, extended keyboard, selection). Without this data, the maintenance cost of the custom renderer cannot be evaluated against actual usage. A hybrid approach (Ink + targeted escape hatches) could achieve 70-85% of differentiation at 50-70% lower maintenance cost.

**Challenge 6: Bus factor risk**
The renderer requires sustained systems-level terminal expertise. If institutional knowledge leaves the team, compatibility regressions become harder to diagnose and protocol drift goes undetected.

## Summary

Claude Code's terminal renderer is a **production-grade, heavily-extended fork of vadimdemedes/ink** that has been rebuilt at the cell level with performance characteristics closer to Ratatui than to stock Ink. The architecture is well-engineered: integer-interned cell buffers, structured diff/patch pipeline, double-buffering, synchronized output, and a full browser-style event system.

The decision to build custom was driven by npm Ink's hard ceiling: no mouse, no selection, no scrolling, no hyperlinks, no alt-screen control. These are table-stakes features for an AI CLI with streaming output and rich interactivity.

The trade-off is clear: **full control at the cost of full ownership**. The renderer carries two forked upstreams (Ink + Yoga), fragile React internal dependencies, and documented performance fallbacks under common conditions (tmux, selection, resize). Long-term sustainability depends on team commitment to terminal systems expertise and a testing program that covers cross-terminal compatibility, protocol correctness, and performance regression.

## Possible Next Steps

- **If this analysis raises design decisions to resolve** → `/ae:discuss 013-ink-rendering-architecture` to debate trade-offs (e.g., hybrid approach, telemetry requirements, testing strategy)
- **If the architecture is understood and next action is clear** → `/ae:plan` to define implementation steps (e.g., cross-terminal test matrix, telemetry instrumentation)
- **If specific subsystems need deeper investigation** → `/ae:trace` for rendering pipeline hot paths or `/ae:think` for performance analysis
