# GSD Deep Dive: Wave-Based Execution & Task Grouping

**March 2026**

This document analyzes GSD's wave execution model — how tasks are grouped, ordered, and dispatched — and compares it with Rein's sequential model and Gas Town's concurrent dispatch.

---

## 1. Wave Execution Overview

GSD groups plans within a phase into "waves" based on dependencies. Plans within a wave run in parallel (if `parallelization.enabled`). Waves run sequentially.

```
Wave 1 (parallel)          Wave 2 (parallel)          Wave 3
┌─────────┐ ┌─────────┐    ┌─────────┐ ┌─────────┐    ┌─────────┐
│ Plan 01 │ │ Plan 02 │ →  │ Plan 03 │ │ Plan 04 │ →  │ Plan 05 │
│ User    │ │ Product │    │ Orders  │ │ Cart    │    │ Checkout│
│ Model   │ │ Model   │    │ API     │ │ API     │    │ UI      │
└─────────┘ └─────────┘    └─────────┘ └─────────┘    └─────────┘
```

**Source:** README.md, `get-shit-done/workflows/execute-phase.md`

---

## 2. How Waves Are Determined

Each PLAN.md file has YAML frontmatter with wave assignment and dependency declarations:

```yaml
---
phase: 3
plan: 2
wave: 2
depends_on: [3-1]
autonomous: true
files_modified: [src/orders/api.ts, src/orders/types.ts]
---
```

The planner agent (gsd-planner) assigns waves during `/gsd:plan-phase` based on:
- **Dependency analysis:** Plans that depend on other plans go in later waves
- **File conflict detection:** Plans modifying the same files go in the same wave (sequential) or different waves
- **Vertical slice preference:** GSD's planner favors vertical slices (feature end-to-end) over horizontal layers (all models, then all APIs)

Wave assignment is done by the LLM planner, not by code. There is no topological sort or dependency graph solver — the planner agent reads the requirements and creates waves based on its understanding of dependencies.

The plan-checker agent (gsd-plan-checker) verifies wave assignments:
- Dependencies correctly identified
- Waves assigned for parallel execution
- File conflicts accounted for

If the checker finds issues, a revision loop runs (max 3 iterations) until plans pass verification or the user overrides.

**Source:** `get-shit-done/workflows/plan-phase.md`, `agents/gsd-planner.md`, `agents/gsd-plan-checker.md`

---

## 3. Wave Execution Protocol

From `execute-phase.md`, the wave execution protocol:

### Step 1: Discover and Group Plans
```bash
PLAN_INDEX=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" phase-plan-index "${PHASE_NUMBER}")
```

The CLI tool returns a JSON structure with plans grouped by wave, including metadata (objective, files_modified, task_count, has_summary).

### Step 2: Execute Each Wave

For each wave in sequence:

1. **Describe** what's being built (from plan objectives)
2. **Spawn** executor agents — one per plan:
   ```
   Task(
     subagent_type="gsd-executor",
     model="{executor_model}",
     prompt="Execute plan {plan_number}..."
   )
   ```
3. **Wait** for all agents in wave to complete
4. **Spot-check** each SUMMARY.md:
   - Verify first 2 files from `key-files.created` exist on disk
   - Check `git log` for commits matching the plan
   - Check for `## Self-Check: FAILED` marker
5. **Report** completion per wave
6. **Handle failures** — user chooses: retry plan, continue, or stop

### Step 3: Post-Wave Checkpoints

Plans marked `autonomous: false` require user interaction between waves. The orchestrator presents checkpoint details and spawns continuation agents after user response.

### Step 4: Phase Verification

After all waves complete, spawn a verifier agent:
```
Task(
  subagent_type="gsd-verifier",
  model="{verifier_model}",
  prompt="Verify phase goal achievement..."
)
```

The verifier checks the codebase against the phase's goal and requirement IDs.

**Source:** `get-shit-done/workflows/execute-phase.md`

---

## 4. Parallelization Configuration

From `config.json`:

```json
{
  "parallelization": {
    "enabled": true,
    "plan_level": true,       // Plans within a wave run in parallel
    "task_level": false,      // Tasks within a plan run sequentially
    "skip_checkpoints": true, // Skip checkpoints in parallel waves
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2
  }
}
```

Key observations:
- **Plan-level parallelism only.** Tasks within a plan always run sequentially. This is a significant constraint — it means parallelism is at the plan granularity, not the task granularity.
- **Max 3 concurrent agents.** Modest compared to Gas Town's 20-30. This is driven by Claude's API rate limits and cost, not by architectural limitation.
- **Minimum 2 plans for parallel.** Single-plan waves run sequentially (no overhead of parallel dispatch).

**Source:** `get-shit-done/templates/config.json`

---

## 5. Comparison: Wave Execution vs Other Models

| Dimension | GSD Waves | Rein (Sequential) | Gas Town (Concurrent) | Ralph (Iterative) |
|-----------|-----------|---------------------|----------------------|-------------------|
| **Dispatch model** | Waves of parallel plans | One task at a time | 20-30 concurrent agents | One iteration at a time |
| **Dependency handling** | LLM-assigned waves with checker verification | Operator-defined task sequence | Mayor decomposes, formula templates | Agent picks one task per iteration |
| **Concurrency** | 1-3 plans per wave | None (sequential) | 20-30 agents per dispatch | None (process restart) |
| **Isolation** | Task tool (fresh subagent context) | Subprocess per task | Git worktree per agent | Process restart |
| **Merge conflicts** | Same-wave plans on separate files (LLM-planned) | N/A (sequential) | Refinery merge queue | N/A (single agent) |
| **Failure handling** | User chooses: retry, continue, stop | Validation commands (pass/fail) | Witness supervision + nudges | Tests pass/fail, iteration cap |
| **Who decides order** | LLM planner + checker | Operator | Mayor agent | Agent reads spec |
| **Cost scaling** | Linear per wave, parallel within | Linear per task | Quadratic (20-30 agents × context) | Linear per iteration |

### Wave Execution as the Middle Ground

GSD's wave model occupies a middle position between Rein's strict sequential execution and Gas Town's aggressive concurrency:

**Advantages over sequential:**
- Independent plans execute simultaneously, reducing wall-clock time
- Dependency analysis prevents ordering errors
- Wave boundaries provide natural synchronization points

**Advantages over fully concurrent:**
- Smaller blast radius than Gas Town (3 concurrent vs 20-30)
- LLM-planned dependency waves reduce merge conflicts
- Human-in-the-loop at wave boundaries (checkpoints)
- Lower cost (fewer concurrent agents)

**Disadvantages:**
- LLM-planned waves may have incorrect dependencies (the checker catches some but not all)
- No merge queue — relies on file-level conflict avoidance rather than merge resolution
- Orchestrator context accumulates across waves
- No automated conflict detection — file conflicts are declared in frontmatter, not detected at runtime

---

## 6. Wave Execution Quality

### Strengths

**1. Plan-checker feedback loop.** The planner creates plans, the checker verifies, and up to 3 revision iterations refine them. This catches dependency errors and coverage gaps before execution.

**2. Atomic commits per task.** Each task within each plan gets its own commit. Git bisect can identify the exact failing task. This is better than both Ralph (bulk commits) and Gas Town (merge-based commits).

**3. Spot-checking summaries.** After each wave, the orchestrator verifies files exist and commits are present. This catches executor hallucinations (claiming files were created when they weren't).

**4. Post-execution verification.** The verifier agent checks goal achievement against requirements — not just "did it compile" but "did it deliver what the phase promised."

### Weaknesses

**1. LLM-planned dependency analysis is fragile.** The wave assignments are created by a language model reading requirements, not by analyzing actual code dependencies. This works for greenfield projects where the planner defines the architecture. It fails for brownfield projects where real dependency graphs are complex and implicit.

**2. No runtime conflict detection.** If two parallel executors modify the same file (due to incorrect wave assignment), the result is a git conflict with no automated resolution. Gas Town handles this with a dedicated Refinery merge queue. GSD relies on correct planning to avoid it.

**3. Checkpoint complexity.** The checkpoint protocol (continuation agents, structured state return, auto-mode handling) is the most complex part of GSD's codebase. The continuation agent pattern (spawn fresh agent with previous task state rather than resuming) is pragmatic but adds latency and context overhead.

**4. All-or-nothing wave progression.** If one plan in a wave fails, the user must decide whether to retry or continue. There's no partial wave re-execution — the entire wave must be re-run or the failure accepted.

---

## 7. Implications for the Rein

### Adopt (When Multi-Task Mode Exists): Wave Grouping Concept

When rein supports multi-task execution, grouping tasks into dependency-ordered waves is the right middle ground between sequential and concurrent. Rein should:
- Support explicit dependency declarations in task definitions (a `depends_on` field)
- Group tasks with no dependencies into parallel waves
- Execute waves sequentially, tasks within waves in parallel
- Validate after each wave before proceeding

### Skip: LLM-Planned Dependencies

Rein's tasks are operator-defined. Dependencies should be operator-declared, not LLM-inferred. The planner + checker loop adds complexity and fragility that isn't needed when the operator knows the dependency graph.

### Study: Spot-Check Pattern

GSD's post-wave spot-checking (verify files exist, verify commits present) is a lightweight validation that catches hallucinated summaries. Rein's validation commands serve a similar purpose but are more formal (exit codes, pass/fail scoring). Consider adding file-existence checks as built-in validation for tasks that declare expected outputs.

---

## Sources

- glittercowboy/get-shit-done: README.md, `get-shit-done/workflows/execute-phase.md`, `get-shit-done/workflows/plan-phase.md`
- `agents/gsd-executor.md`, `agents/gsd-planner.md`, `agents/gsd-plan-checker.md`
- `get-shit-done/templates/config.json` — parallelization config
- Rein docs: TASKS.md, ARCHITECTURE.md (sequential execution, multi-agent future)
- Prior deep dives: gas_town_deep_dive/00_synthesis.md (concurrent dispatch), ralph_wiggum_deep_dive/00_synthesis.md (iterative model)
