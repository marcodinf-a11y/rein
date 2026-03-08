# T3Code Deep Dive: High-Value Proposals

**March 2026**

Assessment of what Rein should adopt from T3Code, what to skip, and why the skip list matters more than the adopt list.

---

## Verdict: Worth Studying, Not Worth Envying

T3Code is a well-architected TypeScript GUI wrapper for coding agents. ~2.5k stars, v0.0.4, 889 commits. Built by Ping.gg with Effect-based DI, SQLite event sourcing, React 19 frontend, Electron desktop app.

T3Code and Rein solve different problems. T3Code makes agent interaction prettier (UI, persistence, replay). Rein makes agent execution better (monitoring, evaluation, intervention). T3Code has zero context pressure monitoring and zero quality evaluation — Rein's entire reason for existing.

---

## Adopt: 3 Patterns

### 1. Contracts-First Event Taxonomy (highest value)

**What:** T3Code defines 47+ runtime events and 20 orchestration events as pure types with zero business logic. Every event has a discriminated `type` field enabling exhaustive pattern matching.

**Why it matters for Rein:** Rein currently uses ad-hoc logging. Structured events in `rein/events.py` (~15 types, not 67+) would enable post-hoc debugging ("why was this session killed?"), `jq` queries on JSONL event logs, and future dashboard/webhook integration.

**Confidence:** High — this converges with the Ralph orchestrator proposal for JSONL event streams. Two independent deep dives arriving at the same conclusion.

**Target doc:** SESSIONS.md (new section). See [proposal 02](proposals/02_canonical_event_taxonomy.md).

**Effort:** Medium. Define types, add emit calls to runner/monitor/gate.

### 2. Stream-Based Adapter Interface (clean refactor)

**What:** T3Code's adapters expose `streamEvents: Stream<ProviderRuntimeEvent>` — typed events, not raw stdout lines. The monitor consumes the stream without knowing which agent produced it.

**Why it matters for Rein:** Rein's current `AgentAdapter` has only `name`, `is_available()`, `run()`. The monitor parses stdout directly, coupling monitoring logic to each agent's output format. Adding `stream_events()` pushes format-specific parsing into the adapter where it belongs. The monitor becomes agent-agnostic. Adding a new agent means implementing one parser method, not touching monitor code.

Also add `check_health()` for structured pre-flight checks — `rein list agents` with version info instead of just available/not-found.

**Target doc:** ARCHITECTURE.md (modify AgentAdapter protocol). See [proposal 03](proposals/03_provider_adapter_contract.md).

**Effort:** Low. `stream_events` is a refactor of existing parsing logic; `check_health` is a simple extension.

### 3. Worktree Lifecycle Specification (fill a spec gap)

**What:** Rein's TASKS.md documents worktree as a workspace type in 3 lines. T3Code's production worktree implementation (889 commits, actively used) reveals concrete needs:

- Configurable branch prefix — T3Code hardcoded `t3code/` and it's now issue #272 because teams can't change it. Rein should make it configurable from day one.
- Pre-flight validation — does the branch already exist? Is there dirty state? These checks prevent cryptic git errors.
- Cleanup strategy — keep branch on success/failure (user reviews and merges), delete on adapter error (no useful work), `rein clean` for housekeeping.
- Known gotchas — shared `.git` directory conflicts with parallel worktrees, submodule initialization, port conflicts between concurrent worktrees.

**Target doc:** TASKS.md (expand worktree subsection). See [proposal 04](proposals/04_worktree_session_isolation.md).

**Effort:** Medium. Spec work that prevents debugging surprises at implementation time.

---

## Skip: Everything Else

The skip list matters more than the adopt list. These are the things that look impressive in T3Code's architecture but would be wasted effort in Rein.

| Pattern | Why Skip |
|---------|----------|
| **Effect library** | Steep learning curve, tiny ecosystem, LLMs can't reason about it. Python protocols + constructor injection are fine for Rein's complexity level. |
| **Full CQRS / Event Sourcing** | ~2000+ lines of infrastructure for replay and projections. Rein is one task, one agent, one report. A JSONL event log gets 80% of the debugging value at 10% of the complexity. |
| **WebSocket RPC protocol** | 18-method protocol serving a web UI that Rein doesn't have and shouldn't build. |
| **Electron desktop app** | Rein is infrastructure, not interface. |
| **Session state machine (7 states)** | Rein has kill-or-complete. Adding idle/starting/interrupted/stopped/error/ready is premature until session resume (R2) adds real state transitions. |
| **`interrupt()` / `resume()` / `rollback()` adapter methods** | Rein uses subprocess signals (SIGTERM/SIGINT). These adapter methods need event sourcing infrastructure to work properly — infrastructure Rein doesn't have. |
| **204KB ChatView.tsx** | A warning, not a pattern. This is what happens without component decomposition. |
| **SQLite with 13 migrations** | Event sourcing persistence layer. Rein writes JSON files. The file-per-report model is simpler and sufficient for CLI workflows. |

---

## Where Rein Is Already Ahead

This is the part that matters most — confirming Rein's core value is not something T3Code provides or threatens:

| Capability | T3Code | Rein |
|-----------|--------|------|
| Context pressure monitoring | Nothing | Real-time zone-based (green/yellow/red) |
| Automated quality evaluation | Nothing | Structured binary evaluation (validation commands) |
| Context enforcement | Can't kill agents at zone boundaries | Zone-based kill (SIGTERM → SIGKILL) with wrap-up |
| Workspace flexibility | Worktree only | Tempdir, worktree, copy |
| Token accounting | None | Normalized cross-provider tracking |
| External monitoring | Orchestration events consume agent context | Zero context overhead — monitoring is external |
| Brownfield support | Worktree only | 3 workspace types, brownfield by design |

---

## Biggest Risk

The biggest risk from this deep dive isn't missing something from T3Code. It's getting distracted by their infrastructure and building event sourcing, CQRS, Effect-based DI, or a web UI when Rein doesn't need any of it.

T3Code's architecture is sophisticated because it solves a GUI problem (real-time event-driven UI state). Rein solves a monitoring problem (read stream, check pressure, kill if needed). The monitoring problem needs ~15 event types, a stream parser per adapter, and a well-specified worktree lifecycle. That's it.

---

## Priority Order

| # | What | Target | Effort | When |
|---|------|--------|--------|------|
| 1 | Worktree lifecycle spec | TASKS.md | Medium | Now (P0) — blocks implementation |
| 2 | `stream_events()` + `check_health()` on AgentAdapter | ARCHITECTURE.md | Low | Now (P1) — clean refactor |
| 3 | ~15 structured event types + JSONL log | SESSIONS.md | Medium | Now (P1) — enables debugging |
| 4 | Ecosystem positioning (T3Code in landscape) | ARCHITECTURE.md | Low | Now — documentation only |
| 5 | Causation tracking on events | REPORTS.md | Low | R2 — when session resume lands |
| 6 | Optional event store | — | High | R3 — only if dashboard/parallel dispatch demands it |

---

## Sources

- [00 Synthesis](00_synthesis.md) — cross-document findings
- [01 Core Architecture](01_core_architecture.md) — monorepo, tech stack
- [02 Contracts & Event System](02_contracts_event_system.md) — 67+ event types
- [03 Provider Adapter Pattern](03_provider_adapter_pattern.md) — adapter interface, `streamEvents`
- [04 Orchestration Engine](04_orchestration_engine.md) — CQRS/ES, SQLite persistence
- [05 Workspace Management](05_workspace_management.md) — worktree lifecycle, issue #272
- [06 Frontend & Desktop](06_frontend_desktop.md) — React/Electron, WebSocket protocol
- [07 Critical Analysis](07_critical_analysis.md) — strengths, weaknesses, adopt/skip
- [Proposals](proposals/00_overview.md) — 5 enrichment proposals with proposed section text
