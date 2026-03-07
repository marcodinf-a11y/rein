# Proposal: Wave-Based Multi-Task Execution

**Priority:** Next (when multi-task mode exists) | **Effort:** Medium | **Source:** GSD Deep Dive

---

## Summary

When rein supports multi-task dispatch, implement wave-based execution: group independent tasks into parallel waves based on operator-declared dependencies, execute waves sequentially.

## Rationale

GSD's wave model occupies the right middle ground between Rein's current strict sequential model and Gas Town's aggressive concurrent dispatch (20-30 agents). Waves provide:
- Parallelism for independent tasks (reduced wall-clock time)
- Sequential ordering for dependent tasks (correctness)
- Natural synchronization points for validation between waves
- Moderate blast radius (2-3 concurrent vs 20-30)

Rein should use operator-declared dependencies (not LLM-inferred, as GSD does), keeping the human in control of task ordering.

## Proposed Design

### TaskDefinition Extension

Add `depends_on` to `TaskDefinition`:

```python
@dataclass(frozen=True)
class TaskDefinition:
    # ... existing fields ...
    depends_on: list[str] = field(default_factory=list)  # Task IDs this depends on
```

### JSON Schema

```json
{
    "depends_on": {
        "type": "array",
        "items": { "type": "string" },
        "default": [],
        "description": "Task IDs that must complete before this task can start"
    }
}
```

### Wave Computation

```python
def compute_waves(tasks: list[TaskDefinition]) -> list[list[TaskDefinition]]:
    """Group tasks into dependency-ordered waves using topological sort."""
    # Tasks with no dependencies → Wave 1
    # Tasks depending only on Wave 1 tasks → Wave 2
    # etc.
```

This is a standard topological sort — no LLM involved.

### Execution Model

```
Wave 1 (parallel):  task-a, task-b, task-c    (no dependencies)
  ↓ validate all pass
Wave 2 (parallel):  task-d (depends: a), task-e (depends: b)
  ↓ validate all pass
Wave 3 (sequential): task-f (depends: d, e)
```

### Task Directory Convention

```
tasks/pipeline/
    01_setup.json          # wave 1, no deps
    02_models.json         # wave 1, no deps
    03_api.json            # wave 2, depends_on: [01_setup, 02_models]
    04_tests.json          # wave 3, depends_on: [03_api]
```

### Inter-Wave Validation

Run validation commands after each wave completes, before starting the next. If any task in a wave fails validation, halt the pipeline (do not start dependent tasks).

## Why Not LLM-Planned Dependencies

GSD uses an LLM planner to assign waves, then a checker agent to verify. This adds:
- Cost (2+ LLM calls for planning + checking)
- Fragility (LLM may misidentify dependencies)
- Latency (planner-checker revision loop)

Rein's tasks are human-written. The operator knows the dependency graph. Declaring dependencies explicitly is simpler, cheaper, and more reliable.

## Target Documents

- TASKS.md — add `depends_on` field to schema
- ARCHITECTURE.md — add wave execution to multi-agent section
- SESSIONS.md — add inter-wave validation to session lifecycle

## Verification

- [ ] `depends_on` field added to TaskDefinition
- [ ] Wave computation handles: no deps, linear chain, diamond deps, circular deps (error)
- [ ] Parallel execution within waves
- [ ] Validation between waves
- [ ] Pipeline halts on wave failure
