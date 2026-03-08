# T3Code Deep Dive — Synthesis

## Purpose

This document consolidates findings from seven specialist analyses of T3Code (github.com/pingdotgg/t3code) and its relevance to Rein. It cross-references conclusions across documents and extracts actionable insights.

For the detailed analysis, see the individual reports:
- [01 Core Architecture](01_core_architecture.md) — Monorepo structure, tech stack, build system, entry points
- [02 Contracts & Event System](02_contracts_event_system.md) — 47+ runtime events, 20 orchestration events, schemas
- [03 Provider Adapter Pattern](03_provider_adapter_pattern.md) — Adapter interface, Codex/Claude Code implementations
- [04 Orchestration Engine](04_orchestration_engine.md) — Event sourcing, CQRS, projections, persistence
- [05 Workspace Management](05_workspace_management.md) — Git worktrees, session isolation, branch naming
- [06 Frontend & Desktop](06_frontend_desktop.md) — React 19/Zustand web app, Electron desktop, WebSocket protocol
- [07 Critical Analysis](07_critical_analysis.md) — Honest assessment, what Rein should adopt/adapt/skip

Enrichment proposals:
- [Proposal Index](proposals/00_overview.md)
- [01 Ecosystem Positioning](proposals/01_t3code_ecosystem_positioning.md)
- [02 Canonical Event Taxonomy](proposals/02_canonical_event_taxonomy.md)
- [03 Provider Adapter Contract](proposals/03_provider_adapter_contract.md)
- [04 Worktree Session Isolation](proposals/04_worktree_session_isolation.md)
- [05 Orchestration Event Sourcing](proposals/05_orchestration_event_sourcing.md)

---

## What T3Code Is

T3Code is a TypeScript monorepo (Bun + Turborepo) that provides a web/desktop GUI for coding agents. Created by Ping.gg, it has ~2.5k stars, 292 forks, 889 commits, and is at v0.0.4. It wraps agent CLIs (Codex, Claude Code) behind a provider adapter layer and exposes them through a React 19 web app and Electron desktop client. Its architecture is built on the Effect library for dependency injection and service composition, with SQLite-backed event sourcing for persistence.

```
Desktop (Electron)  /  Web (React 19 + Zustand)
            |
     WebSocket RPC (18 methods, 3 push channels)
            |
     Server (Node + Effect)
       - OrchestrationEngine (CQRS / Event Sourcing)
       - ProviderAdapterRegistry (Codex, Claude Code)
       - SQLite Event Store (13 migrations)
       - Git Worktree Manager
            |
     Packages
       - contracts (47+ runtime events, 20 orchestration events, pure schemas)
       - shared (git utilities)
```

---

## Cross-Document Consensus (High Confidence)

### 1. Contracts-First Design Is T3Code's Strongest Pattern

Documents 01, 02, 03, and 07 independently highlight the `packages/contracts` approach. Zero runtime logic, zero business dependencies — only Effect Schema types defining the entire event taxonomy and adapter interfaces. This is the pattern most directly applicable to Rein's `models.py`.

### 2. The Event Taxonomy Is Comprehensive but Over-Engineered for CLI Use

Document 02 catalogs 47+ runtime events and 20 orchestration events. Document 07 notes this richness serves T3Code's GUI (every event drives a UI update) but is excessive for a CLI tool. Rein needs ~15-20 events for its monitoring domain, not 67+.

### 3. The Provider Adapter Pattern Is Well-Designed

Document 03 shows `ProviderAdapterShape<E>` with 11 methods and a `streamEvents` stream. The interface cleanly separates lifecycle management (start/stop/interrupt) from observation (event streaming). Rein's `AgentAdapter` protocol should adopt the stream-based observation pattern while keeping a simpler method set.

### 4. Event Sourcing Adds Power but Significant Complexity

Document 04 details the full CQRS/ES stack: Decider, Projectors, Reactors, SQLite event store. Document 07 concludes this is premature for Rein R1. The JSONL event log (Proposal 05) captures 80% of the debugging value at 10% of the complexity.

### 5. Git Worktree Isolation Is Production-Validated

Document 05 confirms T3Code's worktree pattern works in practice (each thread gets its own worktree with `t3code/{hex}` branches). Document 07 notes the hardcoded prefix (issue #272) as a design mistake Rein should avoid. Rein's multi-workspace-type approach (tempdir/worktree/copy) is more flexible.

### 6. Claude Code Support Is Real but New

Document 03 confirms PR #179 merged the `ClaudeCodeAdapter` using `@anthropic-ai/claude-agent-sdk`. This corrects the old single-file assessment which said "Claude Code planned." However, the Codex adapter is more mature (original implementation), so Claude Code support should be considered early-stage.

### 7. T3Code and Rein Are Complementary, Not Competitive

All documents converge on this conclusion. T3Code provides GUI + persistence + replay. Rein provides monitoring + evaluation + intervention. Neither tool does what the other does. A T3Code user running agents through Rein would get both layers of value.

---

## Where Rein Is Ahead

| Capability | T3Code | Rein |
|-----------|--------|------|
| Context pressure monitoring | None | Real-time zone-based (green/yellow/red) |
| Automated quality evaluation | None | Structured JSON quality gate |
| Context enforcement | None | Zone-based kill (SIGTERM → SIGKILL) |
| Workspace flexibility | Worktree only | Tempdir, worktree, copy |
| Brownfield support | Limited (worktree only) | Multi-workspace by design |
| Token accounting | None | Normalized cross-provider tracking |
| CLI-first simplicity | GUI-first | CLI-first, no Electron/web overhead |

## Where T3Code Is Ahead

| Capability | T3Code | Rein |
|-----------|--------|------|
| Event taxonomy | 67+ typed events | Not yet implemented |
| Adapter interface | 11 methods + stream | Basic start/stop/parse |
| Persistence | SQLite event store, full replay | Structured JSON reports |
| UI | React web + Electron desktop | CLI only |
| Multi-provider | Codex + Claude Code (merged) | Claude Code only (R1) |
| Session state machine | 7 states with transitions | Not yet designed |

---

## Top 5 Actionable Takeaways for Rein

1. **Define a canonical event taxonomy in `rein/events.py`** — Discriminated union types for session, pressure, validation, and gate events. ~15-20 types, not 67+. This is the highest-value pattern from T3Code. (Proposal 02)

2. **Add `stream_events()` to AgentAdapter** — Each adapter should emit typed events, not raw stdout lines. This decouples monitoring from parsing and enables adapter-specific event mapping. (Proposal 03)

3. **Make worktree branch prefix configurable** — Learn from T3Code's issue #272. Use `rein/` as default with override in `rein.toml`. (Proposal 04)

4. **Emit a JSONL event log during execution** — Not full event sourcing, just structured append. Enables post-hoc analysis without CQRS infrastructure. (Proposal 05)

5. **Position Rein as infrastructure, not interface** — T3Code shows there's a market for agent GUIs. Rein is the monitoring/enforcement layer that sits beneath any interface (CLI, GUI, or API). Update landscape docs accordingly. (Proposal 01)

---

## Corrections from Previous Assessment

The old single-file `t3code_deep_dive.md` contained several inaccuracies now corrected:

| Claim in Old File | Correction |
|-------------------|------------|
| "Claude Code planned" | Claude Code adapter merged (PR #179) |
| "45 event types" | 47+ runtime events + 20 orchestration events (67+ total) |
| No mention of event sourcing | Full CQRS/ES architecture with SQLite persistence |
| No mention of workspace management | Git worktree isolation per thread |
| No mention of Effect library | Effect-based DI is core to the architecture |
| Simple "patterns to adopt" list | Now expanded into 5 detailed proposals with effort estimates |
