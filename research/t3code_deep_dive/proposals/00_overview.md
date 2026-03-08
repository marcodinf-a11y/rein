# Enrichment Proposals: Overview & Rationale

**March 2026**

Proposals for enriching rein design docs with patterns identified in the T3Code deep dive. Each proposal is self-contained and targets a specific rein doc. No existing files are modified — these are proposed additions for review.

---

## Proposal Index

| # | Proposal | Target Doc | Priority | Source |
|---|----------|-----------|----------|--------|
| [01](01_t3code_ecosystem_positioning.md) | T3Code Ecosystem Positioning | ARCHITECTURE.md | Now | [synthesis](../00_synthesis.md) §7, [critical analysis](../07_critical_analysis.md) §3-4 |
| [02](02_canonical_event_taxonomy.md) | Canonical Event Taxonomy | SESSIONS.md | Now (P1) | [contracts & events](../02_contracts_event_system.md), [synthesis](../00_synthesis.md) §1-2 |
| [03](03_provider_adapter_contract.md) | Provider Adapter Contract Enhancements | ARCHITECTURE.md | Now (P1) | [adapter pattern](../03_provider_adapter_pattern.md), [synthesis](../00_synthesis.md) §3 |
| [04](04_worktree_session_isolation.md) | Worktree Session Isolation | TASKS.md | Now (P0) | [workspace management](../05_workspace_management.md), [synthesis](../00_synthesis.md) §5 |
| [05](05_orchestration_event_sourcing.md) | Orchestration Event Sourcing Patterns | REPORTS.md | R2/R3 | [orchestration engine](../04_orchestration_engine.md), [synthesis](../00_synthesis.md) §4 |

---

## Decision Rationale

### Why These Five

Each proposal addresses a gap where T3Code has a demonstrated, production-tested pattern that rein lacks:

1. **T3Code Ecosystem Positioning** — Rein docs reference Ralph, ralph-orchestrator, Gas Town, and GSD but not T3Code. As the most prominent GUI-first multi-provider agent orchestrator (~2.5k stars), T3Code clarifies Rein's positioning as infrastructure, not interface.

2. **Canonical Event Taxonomy** — T3Code's contracts-first design with 67+ typed events is its strongest architectural contribution. Rein currently uses ad-hoc logging. Defining ~15 structured event types enables debugging, trend analysis, and future integrations without building T3Code's full event sourcing stack.

3. **Provider Adapter Contract Enhancements** — T3Code's `ProviderAdapterShape<E>` with `streamEvents` decouples stream parsing from monitoring. Rein's current `AgentAdapter` protocol lacks typed event streaming and structured health checks. Adding `stream_events()` and `check_health()` is a small refactor with high payoff.

4. **Worktree Session Isolation** — Rein's TASKS.md documents worktree as a workspace type in 3 lines. T3Code's production worktree implementation reveals concrete needs: configurable branch prefix, pre-flight validation, cleanup strategy, known limitations (shared `.git`, submodules). This is the highest-priority proposal — worktree is a primary workspace type.

5. **Orchestration Event Sourcing Patterns** — T3Code's full CQRS/ES validates the principle of structured events over ad-hoc logging. Rein should adopt the principle incrementally (JSONL log → causation tracking → optional event store) without the infrastructure cost.

### What Was Rejected (and Why)

These T3Code patterns were evaluated and rejected:

| Pattern | Why Rejected | Source |
|---------|-------------|--------|
| **Effect library for DI** | Effect adds significant complexity (steep learning curve, small ecosystem, LLM-hostile). Python's Protocol classes and constructor injection are simpler and sufficient for Rein's needs. | [critical analysis](../07_critical_analysis.md) §2, §5 |
| **Full CQRS / Event Sourcing** | Rein's domain is a single task → single agent → single report. Full command/query separation, deciders, projectors, and reactors add ~2000+ lines of infrastructure without proportional value. JSONL event log provides 80% of the debugging value at 10% of the complexity. | [orchestration engine](../04_orchestration_engine.md), [critical analysis](../07_critical_analysis.md) §5 |
| **WebSocket RPC protocol** | Rein is CLI-first. No web UI, no desktop app, no real-time WebSocket consumers. If a dashboard is added later, REST or simple HTTP streaming is simpler than T3Code's 18-method WS protocol. | [frontend & desktop](../06_frontend_desktop.md) §7 |
| **Electron desktop app** | Out of scope. Rein is infrastructure, not interface. | [frontend & desktop](../06_frontend_desktop.md) §9 |
| **Full session state machine** | T3Code defines 7 states with transitions (idle → starting → running → interrupted → stopped/error/ready). Rein's sessions are simpler — single-shot dispatch with kill-or-complete semantics. Defer to R2 when session resume adds state complexity. | [orchestration engine](../04_orchestration_engine.md) §7 |
| **`interrupt()` / `resume()` adapter methods** | Rein uses subprocess signals (SIGTERM/SIGINT), not adapter-level pause/resume. Revisit for R2 session resume. | [adapter pattern](../03_provider_adapter_pattern.md) |
| **`rollback(checkpoint)` adapter method** | Requires event sourcing infrastructure Rein doesn't have and won't build for R1. | [adapter pattern](../03_provider_adapter_pattern.md) |
| **204KB monolith ChatView.tsx pattern** | This is a warning, not a pattern. T3Code's largest component shows what happens without component decomposition. Rein should not replicate this even if a UI is added. | [frontend & desktop](../06_frontend_desktop.md) §8 |

### Mapping to Synthesis Tiers

| Tier | Proposals |
|------|-----------|
| **Now** (low effort, no new infrastructure) | 01 (positioning), 03 (adapter contract), 04 (worktree isolation) |
| **Now** (medium effort, R1 scope) | 02 (event taxonomy) |
| **Later** (R2/R3, design now) | 05 (event sourcing patterns) |
| **Never** | Effect library, full CQRS, WebSocket protocol, Electron app, session state machine, adapter interrupt/resume/rollback |

---

## Relationship to Other Proposals

The T3Code proposals complement findings from all prior deep dives:

| Scope | Prior Proposals | T3Code Proposals |
|-------|----------------|-----------------|
| **Event taxonomy** | Ralph: JSONL event stream (proposal 06) | T3Code: canonical event taxonomy (proposal 02) — more detailed event type design |
| **Adapter interface** | — (no prior adapter proposals) | T3Code: `stream_events()` + `check_health()` (proposal 03) |
| **Workspace isolation** | Gas Town: crash-recoverable task state (proposal 02) | T3Code: worktree lifecycle, branch naming, cleanup (proposal 04) |
| **Persistence patterns** | Gas Town: workflow persistence (proposal 05) | T3Code: event sourcing as future path (proposal 05) |
| **Ecosystem positioning** | Ralph (01), Gas Town (01), GSD (01), Principal Skinner (01) | T3Code (01) — GUI vs. infrastructure distinction |

Key convergence: T3Code's event taxonomy proposal (02) and Ralph's JSONL event stream proposal (06) both arrive at the same conclusion — Rein needs structured event output. The T3Code proposal provides the type design; the Ralph proposal provides the streaming mechanism. They should be implemented together.

---

## Sources

- [T3Code Deep Dive: Synthesis](../00_synthesis.md)
- [T3Code Deep Dive: Core Architecture](../01_core_architecture.md)
- [T3Code Deep Dive: Contracts & Event System](../02_contracts_event_system.md)
- [T3Code Deep Dive: Provider Adapter Pattern](../03_provider_adapter_pattern.md)
- [T3Code Deep Dive: Orchestration Engine](../04_orchestration_engine.md)
- [T3Code Deep Dive: Workspace Management](../05_workspace_management.md)
- [T3Code Deep Dive: Frontend & Desktop](../06_frontend_desktop.md)
- [T3Code Deep Dive: Critical Analysis](../07_critical_analysis.md)
- [Ralph Orchestrator Proposals](../../ralph_orchestrator_deep_dive/proposals/00_overview.md)
- [Gas Town Proposals](../../gas_town_deep_dive/proposals/00_overview.md)
- Rein design docs: ARCHITECTURE.md, SESSIONS.md, TASKS.md, REPORTS.md, AGENTS.md, TOKENS.md
