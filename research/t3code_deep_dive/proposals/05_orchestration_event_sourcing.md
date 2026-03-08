# Proposal: Orchestration Event Sourcing Patterns

**Target:** REPORTS.md (new section: "Event Log")
**Priority:** R2/R3 (design now, implement later)
**Effort:** Low (R1 JSONL log), Medium (R2 causation), High (R3 event store)

---

## Proposed Section: "Event Log (Planned)"

The following section is proposed for addition to REPORTS.md, after the "Output Directory" section.

---

### Event Log (Planned)

> **Status: Planned for R2.** R1 produces JSON reports only. The event log adds continuous observability.

Each task execution can produce a companion JSONL event log alongside the JSON report. The event log captures every monitoring decision point — not just the final result.

```
results/
    fizzbuzz-001_20260307_100000.json           # Report (existing)
    fizzbuzz-001_20260307_100000.events.jsonl    # Event log (new)
```

#### Event Log vs. Event Sourcing

The event log is a **write-ahead log**, not an event store. The distinction matters:

| Aspect | Rein Event Log (R1-R2) | T3Code Event Store |
|--------|----------------------|-------------------|
| Purpose | Debugging, analysis | System of record |
| Replay | `jq` queries only | Full state reconstruction |
| Schema | Append-only JSONL | SQLite with 13 migrations |
| Projections | None | Materialized read models |
| CQRS | No separation | Full command/query separation |
| Complexity | ~50 lines of emit code | ~2000+ lines of infrastructure |

Rein gets 80% of the debugging value at 10% of the complexity by writing structured JSONL without building event sourcing infrastructure.

#### R2: Causation Tracking

When session resume is added (R2), events gain causation fields:

```python
@dataclass(frozen=True)
class ReinEventBase:
    event_id: str
    session_id: str
    task_id: str
    timestamp: str
    sequence: int                    # Monotonic ordering within session
    causation_event_id: str | None   # What caused this event
```

This enables causal chain analysis: "the kill was caused by this zone change, which was caused by this pressure reading." Adapted from T3Code's `OrchestrationEvent.causationEventId` pattern but without the full event sourcing machinery.

#### R3: Optional Event Store (If Needed)

If Rein grows to support parallel dispatch or a web dashboard, consider:
- SQLite append-only event table
- Simple projections for dashboard queries
- Replay for crash recovery

But only if the use case demands it. The JSONL log from R1/R2 provides sufficient value for CLI-first workflows.

---

## Rationale

T3Code implements full CQRS/Event Sourcing: commands → decider → events → store → projectors → read models. This gives T3Code full replay, time-travel debugging, audit trails, and crash recovery. It also adds ~2000+ lines of infrastructure and a steep learning curve (Effect library).

For Rein R1, this is premature. Rein's domain is simpler: one task, one agent, one sandbox, one report. Full CQRS adds no value here.

However, T3Code's event sourcing validates two principles Rein should adopt incrementally:
1. **Structured events are better than ad-hoc logs** — queryable, composable, machine-readable
2. **Causation chains enable debugging** — "why did this happen?" is answerable from the event stream

The phased approach (R1: JSONL log, R2: causation fields, R3: optional event store) captures these principles without the infrastructure cost.

## Source References

- [T3Code Deep Dive: Orchestration Engine](../04_orchestration_engine.md) — CQRS/ES pattern, Decider, Projectors, Reactors
- [T3Code Deep Dive: Synthesis](../00_synthesis.md) — §4 (event sourcing adds power but complexity)
- [T3Code Deep Dive: Critical Analysis](../07_critical_analysis.md) — §5 (skip full CQRS for R1)
- [Ralph Orchestrator Proposals: JSONL Event Stream](../../ralph_orchestrator_deep_dive/proposals/06_event_stream.md) — complementary proposal
- REPORTS.md — current report format (summary only, no event log)
