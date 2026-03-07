# Proposal: Workflow Persistence (Molecule Pattern)

**Target:** TASKS.md (new section)
**Priority:** Next (when multi-task mode is built)
**Effort:** Medium

---

## Proposed Section: "Workflow Persistence (Planned)"

The following section is proposed for addition to TASKS.md, after the "Task Sequences" section (if present) or after "Task Definition."

---

### Workflow Persistence (Planned)

> **Status: Planned.** Requires multi-task sequential execution.

A workflow is a sequence of tasks with dependencies, checkpoints, and durable state. Workflows survive crashes, context pressure kills, and rein restarts — rein resumes from the last checkpoint rather than re-executing completed tasks.

#### Workflow Definition

```json
{
    "workflow_id": "feature-jwt-auth",
    "tasks": [
        {"task_id": "setup-db", "depends_on": []},
        {"task_id": "add-auth-api", "depends_on": ["setup-db"]},
        {"task_id": "add-auth-frontend", "depends_on": ["setup-db"]},
        {"task_id": "add-auth-tests", "depends_on": ["add-auth-api", "add-auth-frontend"]},
        {"task_id": "update-dockerfile", "depends_on": ["add-auth-tests"]}
    ]
}
```

- `depends_on` defines ordering: tasks with no unsatisfied dependencies are eligible for dispatch
- Independent tasks (`add-auth-api` and `add-auth-frontend`) can execute concurrently when concurrent mode is available, or sequentially in arbitrary order
- A task is only dispatched after all its dependencies have `status=completed` and `validation_passed=true`

#### Workflow State File

Rein persists workflow state in `.rein/workflow_state.json`:

```json
{
    "workflow_id": "feature-jwt-auth",
    "status": "in_progress",
    "started_at": "2026-03-07T10:00:00Z",
    "tasks": {
        "setup-db": {"status": "completed", "validation_passed": true, "commit": "abc1234"},
        "add-auth-api": {"status": "completed", "validation_passed": true, "commit": "def5678"},
        "add-auth-frontend": {"status": "in_progress"},
        "add-auth-tests": {"status": "pending"},
        "update-dockerfile": {"status": "pending"}
    },
    "last_checkpoint": "2026-03-07T10:15:00Z",
    "cumulative_tokens": {"input": 150000, "output": 38000}
}
```

#### Crash Recovery

On rein restart:
1. Check for `.rein/workflow_state.json` in the sandbox
2. If found with `status=in_progress`, resume the workflow
3. Skip tasks with `status=completed` and `validation_passed=true`
4. Resume the in-progress task from its last checkpoint (see [proposal 02](02_crash_recoverable_task_state.md))
5. Continue dispatching pending tasks in dependency order

#### Checkpoints

Rein commits workflow state at these points:
- Task dispatch (new task starts)
- Task completion (pass or fail)
- Validation result (inter-session validation, per [ralph-orchestrator proposal 03](../../ralph_orchestrator_deep_dive/proposals/03_inter_session_validation.md))
- Stagnation detection signal (per [ralph-orchestrator proposal 02](../../ralph_orchestrator_deep_dive/proposals/02_stagnation_detection.md))

#### Scope Constraints

- **Maximum depth: 1 level.** Workflows are flat sequences with dependencies — no nested workflows or sub-workflows. If a task is too complex, the operator breaks it into more tasks.
- **No loops.** Dependencies form a DAG (directed acyclic graph). Cycles are rejected at definition time.
- **No dynamic task creation.** The task list is fixed at workflow start. Agents cannot add tasks to the workflow.

---

## Rationale

Gas Town's Molecules are durable workflow instances — directed graphs of Beads with dependencies, gates, and crash recovery. When an agent crashes mid-Molecule, the next session reads the Molecule state and resumes from the last completed step. Gas Town calls this "nondeterministic idempotence" — different sessions may take different paths, but the outcome converges because the workflow definition and acceptance criteria persist in git.

Rein currently has no workflow concept. Multi-task execution (when built) will be a flat list of tasks executed sequentially. This means:
- If rein crashes after completing 3 of 5 tasks, all 5 must re-execute
- There is no way to express dependencies between tasks
- Token budgets cannot be tracked cumulatively across a workflow

The proposed design is drastically simpler than Gas Town's MEOW stack:

| Concept | Gas Town | Rein Proposal |
|---------|----------|-----------------|
| Task unit | Bead (JSONL, global registry) | Task (JSON, per-sandbox) |
| Workflow | Molecule (arbitrary graph, loops, gates) | Workflow (DAG, no loops, no gates) |
| Template | Formula → Protomolecule (2 layers) | Not included (see [proposal 06](06_reusable_task_templates.md)) |
| Nesting | Epics (hierarchical Bead trees) | None (flat task list) |
| Persistence | JSONL ledger across all tasks | Single JSON file per workflow |

This avoids Gas Town's over-engineering (5-layer MEOW stack) while providing the core value: crash-recoverable multi-step workflows with dependency ordering.

## Source References

- [Gas Town Architecture](../01_architecture.md) — Section 4 (MEOW stack, Molecules as durable workflows)
- [Gas Town Critical Analysis](../05_critical_analysis.md) — Section 5.1 (crash recovery, nondeterministic idempotence)
- [Gas Town Task Decomposition](../04_task_decomposition.md) — Section 3 (hierarchical task structures), Section 6 (avoid deep hierarchies)
- TASKS.md — current task definition (no workflow concept)
