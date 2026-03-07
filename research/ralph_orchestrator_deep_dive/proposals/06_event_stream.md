# Proposal: JSONL Event Stream

**Target:** REPORTS.md (complementary output)
**Priority:** Next (when multi-task mode is built)
**Effort:** Medium

---

## Proposed Addition: Rein-Level Event Log

The following describes a proposed JSONL event log that complements the existing structured JSON report.

---

### Event Stream

Rein emits a JSONL event log alongside the structured JSON report. Each line is a self-contained JSON object representing a rein-level event during execution. The event log captures the *process* of execution — what happened and when — while the JSON report captures the *outcome*.

#### Event Types

| Event | Emitted When | Key Fields |
|-------|-------------|-----------|
| `session_start` | Agent subprocess launched | `task_id`, `agent`, `model`, `sandbox_path`, `context_window` |
| `stream_parse` | Token usage updated from stream | `task_id`, `input_tokens`, `output_tokens`, `utilization_pct`, `zone` |
| `zone_change` | Context pressure zone transitions | `task_id`, `from_zone`, `to_zone`, `utilization_pct`, `turn_number` |
| `validation_run` | Validation command executed | `task_id`, `command`, `exit_code`, `duration_ms` |
| `session_kill` | Rein terminates agent | `task_id`, `termination_reason`, `zone`, `signal_sent` |
| `session_complete` | Session finished (any reason) | `task_id`, `exit_code`, `termination_reason`, `duration_seconds`, `total_tokens` |
| `stagnation_detected` | Stagnation detector fires (proposal 02) | `task_id`, `stagnation_signal`, `details` |
| `workflow_gate` | Inter-session validation (proposal 03) | `task_id`, `validations_run`, `all_passed`, `last_good_commit` |

#### Event Schema

Every event shares a common envelope:

```json
{"ts": "2026-03-07T14:30:00.123Z", "event": "zone_change", "task_id": "refactor-auth-001", "data": {...}}
```

| Field | Type | Description |
|-------|------|------------|
| `ts` | ISO 8601 string | Timestamp with millisecond precision |
| `event` | string | Event type (from table above) |
| `task_id` | string | Task that generated the event |
| `data` | object | Event-specific payload |

#### Example Event Sequence

```jsonl
{"ts":"2026-03-07T14:30:00.000Z","event":"session_start","task_id":"auth-001","data":{"agent":"claude-code","model":"sonnet","context_window":200000}}
{"ts":"2026-03-07T14:30:05.123Z","event":"stream_parse","task_id":"auth-001","data":{"input_tokens":12000,"output_tokens":800,"utilization_pct":6.4,"zone":"green"}}
{"ts":"2026-03-07T14:30:45.456Z","event":"stream_parse","task_id":"auth-001","data":{"input_tokens":85000,"output_tokens":12000,"utilization_pct":48.5,"zone":"green"}}
{"ts":"2026-03-07T14:31:20.789Z","event":"zone_change","task_id":"auth-001","data":{"from_zone":"green","to_zone":"yellow","utilization_pct":62.3,"turn_number":8}}
{"ts":"2026-03-07T14:31:21.001Z","event":"session_kill","task_id":"auth-001","data":{"termination_reason":"context_pressure","zone":"yellow","signal_sent":"SIGTERM"}}
{"ts":"2026-03-07T14:31:23.500Z","event":"session_complete","task_id":"auth-001","data":{"exit_code":-15,"termination_reason":"context_pressure","duration_seconds":83.5,"total_tokens":124600}}
{"ts":"2026-03-07T14:31:25.100Z","event":"validation_run","task_id":"auth-001","data":{"command":"pytest tests/test_auth.py -v","exit_code":0,"duration_ms":1580}}
```

#### Output Location

```
results/{task_id}_{YYYYMMDD_HHMMSS}.json      # Structured report (existing)
results/{task_id}_{YYYYMMDD_HHMMSS}.events.jsonl  # Event log (new)
```

Same naming convention as the report, with `.events.jsonl` suffix. One event log per run, co-located with its report.

#### Relationship to Reports

The event log **complements** the structured JSON report — it does not replace it.

| Aspect | JSON Report | JSONL Event Log |
|--------|------------|----------------|
| **Captures** | Outcome (what happened) | Process (how it happened) |
| **Granularity** | Per-session snapshot | Per-event timeline |
| **Use case** | Comparison, scoring, CI/CD pass/fail | Debugging, trend analysis, dashboards |
| **Schema stability** | Stable (evaluation contract) | Evolving (new event types can be added) |
| **Machine-readable** | Yes | Yes |

The report answers "did it work?" The event log answers "what happened along the way?"

#### `stream_parse` Frequency

`stream_parse` events are emitted when token counts are updated from the agent's output stream — typically once per agent turn (tool call cycle). For a session with 10 turns, expect ~10 `stream_parse` events. This is not high-frequency data; it will not bloat the log.

In degraded mode (Gemini, Claude with extended thinking), `stream_parse` events are emitted only at completion, so the event log will show `session_start` → `session_complete` with no intermediate pressure data — matching the monitoring capability.

---

## Rationale

Ralph-orchestrator's JSONL event bus (`event_logger.rs`) is the most machine-readable artifact it produces. Each iteration appends events to `.ralph/events-{timestamp}.jsonl` with structured topics, payloads, and timestamps. The RPC API also emits `IterationStart`/`IterationEnd` telemetry for the web dashboard.

Rein currently captures snapshots — a before state and an after state per session, compiled into the JSON report. It does not persist the timeline of what happened *during* execution. This means:

1. **Debugging context pressure kills requires reading raw stream output.** With an event log, `zone_change` events provide a clean timeline of pressure escalation.

2. **Trend analysis across runs is report-level only.** The event log would enable finer-grained analysis: "across 50 runs, yellow-zone transitions happen at turn 7 on average" or "validation command X takes 3x longer than Y."

3. **CI/CD integration is limited to pass/fail.** An event stream could feed real-time dashboards, Slack notifications on zone transitions, or monitoring systems — without parsing the full report.

This is classified as "Next" priority because the event log's value increases significantly with multi-task workflows (where the event timeline spans multiple sessions) and because the current single-session model produces short enough timelines that the JSON report is sufficient.

## Source References

- [Evaluation & Reporting](../04_evaluation_reporting.md) — Section 4 (ralph's machine-readable outputs), Section 7 (what rein could learn)
- [Ralph Orchestrator Synthesis](../00_synthesis.md) — Section 3.3 (JSONL event logging)
- REPORTS.md — current report schema (snapshot model)
- SESSIONS.md — context pressure monitoring protocol (source of events)
