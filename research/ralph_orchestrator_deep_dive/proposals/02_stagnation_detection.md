# Proposal: Stagnation Detection

**Target:** SESSIONS.md (new section)
**Priority:** Next (when multi-task mode is built)
**Effort:** Medium

---

## Proposed Section: "Stagnation Detection (Planned)"

The following section is proposed for addition to SESSIONS.md, after the "Multi-Session Patterns" section.

---

### Stagnation Detection (Planned)

> **Status: Planned.** Requires multi-task sequential execution. The design below is adapted from ralph-orchestrator's three-detector model.

When running multiple sessions sequentially (multi-task workflows), rein tracks failure patterns across sessions to detect stagnation — cases where the agent is stuck and additional sessions will not make progress.

#### Three Detectors

**1. Stale Sessions**

The same validation failure occurs in 3 consecutive sessions on the same task. The agent is repeating the same approach without progress.

Detection: compare `validation_output` across consecutive failed sessions for the same `task_id`. If the failing command and error output are substantively identical 3 times, flag as stale.

```json
{
    "stagnation_signal": "stale",
    "task_id": "refactor-auth-001",
    "consecutive_identical_failures": 3,
    "failing_command": "pytest tests/test_auth.py -v",
    "repeated_error": "FAILED test_auth.py::test_refresh_token - KeyError: 'exp'"
}
```

**2. Oscillation**

A task alternates between passing and failing across sessions — pass/fail/pass/fail. The agent is fixing one thing and breaking another each time.

Detection: track the pass/fail sequence for each `task_id` across sessions. If the pattern contains 3+ alternations (e.g., pass-fail-pass-fail), flag as oscillation.

```json
{
    "stagnation_signal": "oscillation",
    "task_id": "refactor-auth-001",
    "score_sequence": [1.0, 0.0, 1.0, 0.0],
    "sessions": 4
}
```

**3. Consecutive Crashes**

3 consecutive sessions for the same task end with non-zone-kill exits (process crashes, adapter errors, or non-zero exits unrelated to context pressure). The agent or environment is misconfigured.

Detection: track `termination_reason` and `exit_code` across sessions. If 3 consecutive sessions end with `termination_reason=error` or unexpected non-zero exit codes (excluding `context_pressure` and `timed_out`), flag as crash loop.

```json
{
    "stagnation_signal": "crash_loop",
    "task_id": "refactor-auth-001",
    "consecutive_crashes": 3,
    "exit_codes": [-11, -11, -11]
}
```

#### Stagnation Response

When any detector fires:

1. **Log the stagnation signal** in the structured report for the current session
2. **Stop further sessions** for the affected task — do not retry
3. **Continue with the next task** in the sequence (stagnation is per-task, not per-workflow)
4. **Include stagnation summary** in the final workflow report

Stagnation detection does not apply to single-session execution (the current default). It activates only in multi-task sequential workflows where the same task may be retried across sessions.

#### Thresholds

| Detector | Default Threshold | Configurable |
|----------|------------------|-------------|
| Stale sessions | 3 identical failures | Yes |
| Oscillation | 3 alternations | Yes |
| Consecutive crashes | 3 non-zone-kill exits | Yes |

---

## Rationale

Ralph-orchestrator implements three stagnation detectors (`loop_state.rs`, `termination.rs`):
- **Stale loop:** same event topic emitted 3 consecutive times
- **Loop thrashing:** abandoned task redispatched 3+ times
- **Consecutive failures:** 5 non-zero exits in a row

These map cleanly to multi-session rein workflows. The adaptation translates ralph's iteration-level concepts to session-level:

| Ralph concept | Rein adaptation |
|--------------|-------------------|
| Same event 3x | Same validation failure 3x |
| Abandoned task redispatched | Task score oscillation (pass/fail/pass) |
| 5 consecutive non-zero exits | 3 consecutive non-zone-kill crashes |

The threshold is lowered from ralph's 5 to 3 for crashes because rein sessions are heavier than ralph iterations (minutes vs. seconds), so the waste cost of each retry is higher.

Rein currently has stagnation detection listed as "planned" in the [Ralph Wiggum synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) and [ralph-orchestrator synthesis](../00_synthesis.md) but with no design specification. This proposal provides a concrete, tested design adapted from ralph's production implementation.

## Source References

- [Iteration Control](../03_iteration_control.md) — Section 3 (stagnation detection details, thresholds, source files)
- [Ralph Orchestrator Synthesis](../00_synthesis.md) — Section 3.1 (transferable pattern), Section 5 (Next tier)
- [Ralph Wiggum Synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) — Section 8 (Next tier: stagnation detection)
