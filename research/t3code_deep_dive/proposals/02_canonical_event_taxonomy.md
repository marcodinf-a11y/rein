# Proposal: Canonical Event Taxonomy

**Target:** SESSIONS.md (new section after "Multi-Session Patterns")
**Priority:** P1 (R1)
**Effort:** Medium

---

## Proposed Section: "Structured Event Log (Planned)"

The following section is proposed for addition to SESSIONS.md, after the "Multi-Session Patterns" section.

---

### Structured Event Log (Planned)

> **Status: Planned.** Design adapted from T3Code's 67+ typed event taxonomy. Rein's domain is narrower — ~15 event types suffice.

During task execution, rein emits structured events to a JSONL log file alongside the JSON report. Events enable post-hoc analysis, debugging ("why was this session killed?"), and future integrations (dashboards, webhooks).

#### Event Base

Every event carries:

```python
@dataclass(frozen=True)
class ReinEventBase:
    event_id: str          # UUID
    session_id: str        # Links to task execution
    task_id: str           # Links to task definition
    timestamp: str         # ISO 8601
    round_id: int | None   # Quality gate round (if applicable)
```

#### Event Types

| Category | Event | Payload | Emitted From |
|----------|-------|---------|--------------|
| **session** | `session.started` | `agent`, `model`, `sandbox_path` | Runner, after subprocess spawn |
| **session** | `session.completed` | `exit_code`, `termination_reason` | Runner, on clean exit |
| **session** | `session.killed` | `signal`, `termination_reason`, `zone` | Monitor, after SIGTERM/SIGKILL |
| **pressure** | `pressure.reading` | `zone`, `utilization_pct`, `tokens_used`, `context_window` | Monitor, each polling interval |
| **pressure** | `pressure.zone_changed` | `from_zone`, `to_zone`, `utilization_pct` | Monitor, on zone transition |
| **turn** | `turn.completed` | `turn_number`, `input_tokens`, `output_tokens` | Monitor, per agent turn |
| **validation** | `validation.started` | `command` | Evaluate, before each command |
| **validation** | `validation.passed` | `command`, `duration_ms` | Evaluate, on exit 0 |
| **validation** | `validation.failed` | `command`, `exit_code`, `stderr` | Evaluate, on non-zero exit |
| **gate** | `gate.round_started` | `round_id` | Quality gate, before evaluation |
| **gate** | `gate.round_completed` | `round_id`, `passed`, `score` | Quality gate, after all checks |
| **wrapup** | `wrapup.started` | `termination_reason` | Runner, on wrap-up entry |
| **wrapup** | `wrapup.commit` | `commit_hash`, `diff_stat` | Runner, after auto-commit |

#### JSONL Output

Events are appended to `results/{task_id}_{timestamp}.events.jsonl` alongside the JSON report:

```jsonl
{"event":"session.started","event_id":"...","session_id":"abc","task_id":"fizzbuzz-001","timestamp":"2026-03-07T10:00:00Z","agent":"claude","model":"sonnet","sandbox_path":"/tmp/rein-abc"}
{"event":"pressure.reading","event_id":"...","session_id":"abc","task_id":"fizzbuzz-001","timestamp":"2026-03-07T10:00:05Z","zone":"green","utilization_pct":23.5,"tokens_used":47000,"context_window":200000}
{"event":"pressure.zone_changed","event_id":"...","session_id":"abc","task_id":"fizzbuzz-001","timestamp":"2026-03-07T10:02:30Z","from_zone":"green","to_zone":"yellow","utilization_pct":62.1}
{"event":"session.killed","event_id":"...","session_id":"abc","task_id":"fizzbuzz-001","timestamp":"2026-03-07T10:02:31Z","signal":"SIGTERM","termination_reason":"context_pressure","zone":"yellow"}
```

#### Querying

Events are plain JSONL — queryable with `jq`:

```bash
# All zone changes for a task
jq 'select(.event=="pressure.zone_changed")' results/fizzbuzz-001_*.events.jsonl

# Why was a session killed?
jq 'select(.event=="session.killed" or .event=="pressure.zone_changed")' results/*.events.jsonl

# Validation failures
jq 'select(.event=="validation.failed")' results/*.events.jsonl
```

#### Relationship to JSON Report

The JSONL event log and the JSON report are complementary:

| Aspect | JSON Report | JSONL Event Log |
|--------|------------|-----------------|
| Granularity | Summary per execution | Every monitoring event |
| Timing | Written once at end | Appended in real-time |
| Use case | Results, scoring, comparison | Debugging, trend analysis |
| Schema | [REPORTS.md](../../../REPORTS.md) | Event types above |

The JSON report remains the primary output. The event log adds observability without changing the report format.

---

## Rationale

T3Code defines 47+ runtime events and 20 orchestration events — its strongest architectural contribution. This richness serves T3Code's GUI (every event drives a UI update) but is excessive for Rein's CLI domain.

Rein currently uses ad-hoc logging for monitoring. The roadmap mentions a JSONL event stream (R3/Future) but doesn't define structured event types. Without a canonical taxonomy, monitoring output is inconsistent and hard to consume programmatically.

The proposed ~15 event types cover Rein's monitoring domain (session lifecycle, pressure readings, zone transitions, validation, wrap-up) without the 67+ types T3Code needs for its GUI. The discriminated `event` field enables exhaustive matching — directly adapted from T3Code's `ProviderRuntimeEvent.type` pattern.

The `session_id` + `task_id` + `round_id` fields provide simpler causation tracking than T3Code's full `causationEventId` / `correlationId` system, while still enabling "why was this killed?" analysis.

## Source References

- [T3Code Deep Dive: Contracts & Event System](../02_contracts_event_system.md) — 47+ runtime events, discriminated types
- [T3Code Deep Dive: Synthesis](../00_synthesis.md) — §1 (contracts-first), §2 (event taxonomy)
- [T3Code Deep Dive: Critical Analysis](../07_critical_analysis.md) — §5 (Rein should adopt event taxonomy)
- [Ralph Orchestrator Proposals: JSONL Event Stream](../../ralph_orchestrator_deep_dive/proposals/06_event_stream.md) — complementary proposal
- SESSIONS.md — current monitoring protocol (no structured event output)
