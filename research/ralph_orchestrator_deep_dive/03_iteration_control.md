# Ralph Orchestrator: Iteration Control & Convergence

**Deep Dive Document 03 | March 2026**

How ralph-orchestrator decides when to continue, stop, or intervene. Compared against Rein's quality gate and token budget.

---

## 1. The Main Loop

The core loop in `run_loop_impl()` (ralph-cli) follows this cycle:

```
┌──────────────────────────────────────┐
│ 1. Check interrupt (Ctrl+C)          │
│ 2. Drain guidance queue (TUI/RPC)    │
│ 3. Check termination conditions      │
│ 4. Run pre-iteration hooks           │
│ 5. Get next hat (event routing)      │
│ 6. Build prompt (scratchpad+tasks)   │
│ 7. Execute agent (PTY subprocess)    │
│ 8. Process output (track cost/fails) │
│ 9. Read events from JSONL            │
│ 10. Check completion event           │
│ 11. Run backpressure gates           │
│ 12. Loop back to step 1              │
└──────────────────────────────────────┘
```

**Continue:** Automatic as long as the EventBus has pending events and no termination condition fires.

**Stop:** When any `TerminationReason` variant triggers.

Source: `run_loop_impl()` in ralph-cli, `event_loop.rs` in ralph-core.

---

## 2. Completion Detection

Ralph uses a **JSONL event system** for completion signaling:

1. The agent runs `ralph emit LOOP_COMPLETE` during execution
2. This appends a JSON line to `.ralph/events-{timestamp}.jsonl`
3. After the iteration, `EventReader` reads new lines (seek-based, not re-reading the full file)
4. `EventLoop` checks if the `completion_promise` topic was emitted

**Validation before accepting completion:**
- **Required events check:** If `required_events` is configured (e.g., `["build.done", "test.pass"]`), all must have been seen during the loop before `LOOP_COMPLETE` is accepted. Otherwise completion is rejected and a `task.resume` event is injected to continue work.
- **Persistent mode:** If `persistent: true`, `LOOP_COMPLETE` is suppressed and the loop stays alive — useful for long-running daemon-like workflows.
- **Task check:** Warns (but does not block) if tasks remain open when completing.

This is significantly more sophisticated than bare Ralph's implicit completion (agent exits = done). It allows the operator to define what "done" actually means.

Source: `event_reader.rs`, `event_loop.rs`, completion validation logic in ralph-core.

---

## 3. Stagnation Detection

Ralph implements three stagnation detectors:

### 3.1 Stale Loop
Tracks `consecutive_same_topic` in `LoopState`. If the same event topic is emitted 3+ consecutive times, the loop terminates with `LoopStale`. This catches the "spinning in place" failure mode where the agent keeps attempting the same task without progress.

### 3.2 Loop Thrashing
Tracks per-task block counts (`task_block_counts`). Tasks blocked 3+ times are added to `abandoned_tasks`. If the planner redispatches abandoned tasks 3+ times (`abandoned_task_redispatches >= 3`), the loop terminates with `LoopThrashing`. This catches the oscillation pattern: fix-A-break-B, fix-B-break-A.

### 3.3 Consecutive Failures
If the agent process exits non-zero 5 consecutive times (configurable via `max_consecutive_failures`), the loop terminates with `ConsecutiveFailures`. This catches hard crashes and misconfigured agents.

### 3.4 Fallback Recovery
When no hats have pending events (agent failed to publish any), a `task.resume` fallback event is injected. After 3 consecutive fallback attempts (`MAX_FALLBACK_ATTEMPTS`), the loop terminates with `Stopped`. This catches the silent failure mode where the agent produces output but no structured events.

Source: `loop_state.rs`, `termination.rs` in ralph-core.

---

## 4. Iteration Limits and Cost Caps

| Limit | Default | Configurable | Notes |
|-------|---------|-------------|-------|
| `max_iterations` | 100 | Yes | Hard cap on loop iterations |
| `max_runtime_seconds` | 14,400 (4h) | Yes | Wall-clock timeout |
| `max_cost_usd` | None | Yes | Cumulative cost from backend metadata |
| `max_consecutive_failures` | 5 | Yes | Consecutive non-zero exit codes |
| Stale loop threshold | 3 | Hardcoded | Same topic 3× in a row |
| Thrashing threshold | 3 | Hardcoded | Abandoned task redispatched 3× |
| Fallback attempts | 3 | Hardcoded | No events published 3× in a row |

**Cost cap accuracy:** Ralph's cost tracking depends entirely on what the backend reports. For Claude with `stream-json`, this is accurate (Anthropic reports actual usage). For other backends, cost data may be unavailable or estimated.

Source: `config.rs`, `loop_state.rs` in ralph-core.

---

## 5. Backpressure Gates

Ralph introduces **backpressure gates** — shell commands run between iterations that must pass before the next iteration starts:

```yaml
gates:
  - name: test
    command: "cargo test"
    on_fail: block
  - name: lint
    command: "cargo fmt --check"
    on_fail: warn
```

Actions on failure:
- `block` — prevents next iteration until gate passes
- `warn` — logs warning, continues anyway

This is functionally similar to Rein's `validation_commands` but runs *between* iterations rather than *after* the final iteration. Ralph validates continuously; rein validates at the end.

Source: `gates.rs` in ralph-core, `ralph.yml` examples.

---

## 6. Comparison with Rein

| Dimension | ralph-orchestrator | Rein |
|-----------|-------------------|----------------|
| **Completion signal** | `ralph emit LOOP_COMPLETE` (JSONL event) | Validation commands (exit code) |
| **Completion validation** | Required events must be seen first | All validation commands must pass |
| **Stagnation detection** | 3 mechanisms (stale, thrashing, fallback) | Not yet implemented (planned) |
| **Oscillation detection** | Yes (thrashing detector) | Not yet implemented (planned) |
| **Cost cap** | `max_cost_usd` (from backend metadata) | Token budget (from stream parsing) |
| **Iteration cap** | `max_iterations` (default 100) | Single session per task (no iteration concept) |
| **Time cap** | `max_runtime_seconds` (default 4h) | `timeout` per task (default 300s) |
| **Mid-iteration control** | Interrupt signal only | Zone-based graceful stop / kill |
| **Inter-iteration validation** | Backpressure gates | Not applicable (single session) |
| **Quality scoring** | Pass/fail (gate exit codes) | Binary (0.0 or 1.0) |

**Key insight:** Ralph trades depth for breadth — many short iterations with inter-iteration validation. Rein trades breadth for depth — one monitored session per task with real-time intervention. Ralph's stagnation detection is more mature than Rein's (which is planned but not implemented). Rein's mid-iteration control is more mature than Ralph's (which has none).

---

## Sources

- github.com/mikeyobrien/ralph-orchestrator (v2.7.0)
- Source files: `run_loop_impl()`, `event_loop.rs`, `loop_state.rs`, `termination.rs`, `gates.rs`
- SESSIONS.md, ARCHITECTURE.md (rein)
- research/ralph_wiggum_deep_dive/04_failure_modes.md
