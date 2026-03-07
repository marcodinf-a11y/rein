# Proposal: Crash-Recoverable Task State

**Target:** TASKS.md (new section)
**Priority:** Now
**Effort:** Low-Medium

---

## Proposed Section: "Task State Persistence"

The following section is proposed for addition to TASKS.md, after the "Task Lifecycle" section.

---

### Task State Persistence

Task state persists across crashes, context pressure kills, and session restarts. Rein writes task state to a structured file in the sandbox at key lifecycle points, enabling resumption from the last checkpoint.

#### State File

Rein maintains a `.rein/task_state.json` file in the sandbox directory:

```json
{
    "task_id": "refactor-auth-001",
    "status": "in_progress",
    "started_at": "2026-03-07T10:00:00Z",
    "last_checkpoint": "2026-03-07T10:12:34Z",
    "session_count": 2,
    "sessions": [
        {
            "session_id": "sess-001",
            "started_at": "2026-03-07T10:00:00Z",
            "ended_at": "2026-03-07T10:05:22Z",
            "termination_reason": "context_pressure",
            "zone_at_exit": "yellow",
            "token_usage": {"input": 45000, "output": 12000},
            "validation_passed": false,
            "commit": "abc1234"
        },
        {
            "session_id": "sess-002",
            "started_at": "2026-03-07T10:06:00Z",
            "ended_at": null,
            "termination_reason": null
        }
    ],
    "last_good_commit": "abc1234",
    "cumulative_tokens": {"input": 90000, "output": 24000}
}
```

#### Lifecycle Checkpoints

Rein writes task state at these points:

| Event | Fields Updated |
|-------|---------------|
| **Task dispatch** | `status=in_progress`, `started_at`, new session entry |
| **Context pressure zone change** | `zone_at_exit` in current session |
| **Session end (any reason)** | `ended_at`, `termination_reason`, `token_usage`, `validation_passed` |
| **Validation pass** | `last_good_commit` (git commit hash of validated state) |
| **Task completion** | `status=completed` or `status=failed` |

#### Crash Recovery

If rein process crashes during execution:

1. On restart, rein checks for `.rein/task_state.json` in the sandbox
2. If found with `status=in_progress`, the task is resumable
3. Rein can resume from `last_good_commit` (the last validated state)
4. Session history is preserved — the new session appends to the `sessions` array
5. Cumulative token usage carries forward for budget enforcement

#### Git-Backed Durability

The state file lives in the sandbox directory. For worktree and copy sandbox modes, rein commits `.rein/task_state.json` alongside agent work at each checkpoint. This ensures task state survives even if rein process is killed (the git commit is atomic).

For tempdir sandbox mode, task state is volatile — a crash loses the state. This is acceptable because tempdir mode is for disposable experimentation.

---

## Rationale

Gas Town's most valuable innovation is Beads — atomic work units persisted as JSONL in git. Tasks survive crashes, context fills, and session restarts because their state is durably written. When an agent crashes mid-task, the next session reads the Bead state and resumes from the last checkpoint.

Rein currently has no equivalent. If rein process crashes during agent execution:
- The agent subprocess is orphaned or killed
- Any uncommitted agent work is lost
- Token usage for the crashed session is unrecorded
- The operator must manually inspect the sandbox and decide whether to restart

Gas Town demonstrates that crash recovery is not a nice-to-have — it's essential for any system running multi-minute agent sessions. Context pressure kills (yellow/red zone interventions) are a form of "controlled crash" that rein already handles, but uncontrolled crashes (process kill, OOM, network failure) are not addressed.

The proposed design is simpler than Gas Town's Beads:
- Single JSON file per task (not a JSONL ledger across all tasks)
- Written at lifecycle checkpoints (not continuously)
- Git-committed for durability (same mechanism as Gas Town)
- No global task registry (each sandbox is self-contained)

This keeps the implementation lightweight while providing the crash recovery that Gas Town proves is necessary.

## Source References

- [Gas Town Architecture](../01_architecture.md) — Section 4 (MEOW stack, Beads as persistent task units)
- [Gas Town Coordination](../02_coordination.md) — Section 2 (worktree isolation, commit-based state preservation)
- [Gas Town Critical Analysis](../05_critical_analysis.md) — Section 5.1 (crash recovery as architectural insight)
- TASKS.md — current task lifecycle (no crash recovery mechanism)
