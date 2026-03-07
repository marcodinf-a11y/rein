# Proposal: Inter-Session Validation

**Target:** SESSIONS.md (new section)
**Priority:** Next (when multi-task mode is built)
**Effort:** Medium

---

## Proposed Section: "Inter-Session Validation (Planned)"

The following section is proposed for addition to SESSIONS.md, after the "Stagnation Detection" section (proposal 02).

---

### Inter-Session Validation (Planned)

> **Status: Planned.** Requires multi-task sequential execution.

In multi-task workflows, rein runs `validation_commands` after each session — not just after the final session. This catches regressions early: if session 2 breaks what session 1 built, rein detects this before sessions 3–N waste compute.

#### Protocol

For a workflow with tasks `[T1, T2, T3, ...]` executed sequentially:

```
Run T1 → Validate T1 → pass? → Run T2 → Validate T1+T2 → pass? → Run T3 → ...
                          │                                  │
                          └─ fail → stop, report             └─ fail → stop, report
```

After each session completes:

1. Run the **current task's** `validation_commands` in the sandbox
2. Run **all prior tasks'** `validation_commands` in the same sandbox (regression check)
3. If any validation fails → stop the workflow, preserve last-good state, report

#### Stop-on-Failure Semantics

When inter-session validation fails:

- **Last-good-state preservation:** The sandbox state from the last session where all validations passed is preserved. Rein commits before each new session starts, so `git reset` to the last-good commit is always available.
- **Report includes:** which task's validation failed, whether it was the current task (new failure) or a prior task (regression), the validation output
- **No automatic retry:** Stagnation detection (proposal 02) handles retries. Inter-session validation is a gate, not a retry mechanism.

#### Cumulative Validation

The key distinction from the current design: validation is **cumulative**, not per-task. After session N, rein validates all tasks T1 through TN, not just TN. This ensures that later tasks do not silently break earlier work.

```json
{
    "inter_session_validation": {
        "session": 3,
        "task_id": "add-rate-limiting",
        "validations_run": [
            {"task_id": "setup-db", "passed": true},
            {"task_id": "add-auth", "passed": true},
            {"task_id": "add-rate-limiting", "passed": false, "output": "..."}
        ],
        "workflow_status": "stopped",
        "last_good_session": 2,
        "last_good_commit": "abc1234"
    }
}
```

#### When Inter-Session Validation Applies

- **Multi-task sequential workflows only.** Single-task execution uses the existing post-completion quality gate unchanged.
- **Only when tasks share a sandbox** (workspace continuity). If each task gets an independent sandbox, there is no shared state to regress.
- **Operator opt-in.** Inter-session validation adds overhead (running prior validations). For workflows where tasks are independent, the operator can disable it.

---

## Rationale

Ralph-orchestrator's backpressure gates (`gates.rs`) run shell commands between iterations:

```yaml
gates:
  - name: test
    command: "cargo test"
    on_fail: block
  - name: lint
    command: "cargo fmt --check"
    on_fail: warn
```

This catches regressions between iterations — functionally identical to the proposed inter-session validation. The key insight from ralph: **validation between execution units is more valuable than validation only at the end**, because it bounds the wasted compute from regressions.

Rein adaptation differs from ralph's gates in two ways:

1. **Cumulative validation.** Ralph gates run the same commands every iteration. Rein runs all prior tasks' validations, catching regressions that a single fixed gate would miss.

2. **Last-good-state preservation.** Ralph's `block` action prevents the next iteration but doesn't preserve state. Rein commits before each session, enabling rollback to the last validated state.

Currently rein validates only after the final session. For a 5-task workflow where task 3 breaks task 1's output, the current design wastes tasks 4 and 5 before discovering the regression. Inter-session validation would catch it immediately after task 3.

## Source References

- [Iteration Control](../03_iteration_control.md) — Section 5 (backpressure gates, YAML config, on_fail semantics)
- [Ralph Orchestrator Synthesis](../00_synthesis.md) — Section 3.2 (transferable pattern), Section 5 (Next tier)
- SESSIONS.md — current quality gate design (post-completion only)
- TASKS.md — `validation_commands` field definition
