# Proposal: Merge Queue for Concurrent Agents

**Target:** SESSIONS.md (new section)
**Priority:** Next (when concurrent multi-agent is built)
**Effort:** High

---

## Proposed Section: "Merge Queue (Planned)"

The following section is proposed for addition to SESSIONS.md, after the "Multi-Agent Patterns" section.

---

### Merge Queue (Planned)

> **Status: Planned.** Requires concurrent multi-agent execution support.

When multiple agents work concurrently on a shared codebase, their changes must be integrated. Rein uses a merge queue — a sequential integration step that reconciles concurrent agent output after execution completes.

#### Architecture

```
Agent A (worktree /a) ─┐
Agent B (worktree /b) ─┼──→ Merge Queue ──→ Integration Branch ──→ Validation
Agent C (worktree /c) ─┘    (sequential)
```

Each concurrent agent operates in an isolated git worktree (see "Execution Isolation"). The merge queue processes completed branches one at a time:

1. Agent completes → commits to feature branch → enters merge queue
2. Queue processes branches in completion order (FIFO)
3. For each branch: attempt rebase onto integration branch
4. If rebase succeeds cleanly → fast-forward merge → run validation
5. If rebase has conflicts → flag for operator resolution
6. If validation fails after merge → revert merge, flag branch

#### Conflict Resolution Policy

Rein does **not** use LLM-based merge resolution. Gas Town's Refinery uses an LLM to resolve merge conflicts, but this approach has no verification step — the LLM may resolve conflicts incorrectly, introducing subtle bugs.

Instead, rein:
- **Clean merges:** Accept automatically, validate with `validation_commands`
- **Conflicting merges:** Flag for operator resolution with diff context
- **Validation failures post-merge:** Revert the merge, report the failure

This is more conservative than Gas Town but more reliable. The operator resolves conflicts with full context rather than trusting LLM judgment on semantic merge correctness.

#### Minimizing Conflicts

Rein reduces merge conflicts through task design rather than resolution:

| Strategy | How |
|----------|-----|
| **Directory partitioning** | Assign tasks that modify different directories to different agents |
| **Module boundaries** | Scope tasks to single modules, avoiding cross-cutting changes |
| **Read-only analysis first** | Run analysis/research tasks concurrently (no writes), then implementation sequentially |
| **Dependency ordering** | Dispatch dependent tasks sequentially, independent tasks concurrently |

#### Merge Queue Report

```json
{
    "merge_queue": {
        "branches_processed": 3,
        "clean_merges": 2,
        "conflicts": 1,
        "validation_failures": 0,
        "conflict_details": [
            {
                "branch": "agent-c/add-rate-limiting",
                "conflicting_files": ["src/middleware.py"],
                "conflicting_with": "agent-a/add-auth",
                "status": "awaiting_operator"
            }
        ]
    }
}
```

---

## Rationale

Gas Town and ralph-orchestrator both implement merge queues for concurrent agent output:

- **Gas Town's Refinery:** Dedicated merge agent (LLM) that resolves conflicts via rebase, including "creatively reimagining implementations" when conflicts are irreconcilable. No verification step. Creates a bottleneck as agent count increases.
- **Ralph-orchestrator:** Worktree per loop with sequential merge queue. Simpler than Gas Town but same basic pattern.
- **Swarm-Tools:** File pattern reservation (DurableLock) to prevent conflicts at write time. More complex but avoids merge-time resolution.

Rein adaptation takes the conservative path:
1. **Worktree isolation** (proven in Gas Town and ralph-orchestrator)
2. **Sequential merge processing** (proven in both)
3. **No LLM merge resolution** (Gas Town's approach is unverified and risky)
4. **Deterministic validation after merge** (Rein's quality gate applied to merged state)

Gas Town's experience shows that the merge queue itself is not the hard problem — the hard problem is minimizing the need for conflict resolution. Rein should invest in task design (directory partitioning, module scoping, dependency ordering) rather than sophisticated conflict resolution.

Maggie Appleton's analysis identifies the Refinery as a key bottleneck: "This offloads human merge responsibility but creates a bottleneck as changes accumulate." Rein should plan for this bottleneck by limiting concurrent agent count (2-5, not 20-30) and optimizing for conflict avoidance over resolution.

## Source References

- [Gas Town Coordination](../02_coordination.md) — Section 3 (Refinery merge queue), Section 5 (what happens on same-file conflicts), Section 7 (implications)
- [Gas Town Critical Analysis](../05_critical_analysis.md) — Section 3 (evidence for/against concurrent multi-agent)
- [Gas Town vs Swarm-Tools](https://gist.github.com/johnlindquist/4174127de90e1734d58fce64c6b52b62) — File coordination comparison
- [Ralph Orchestrator Synthesis](../../ralph_orchestrator_deep_dive/00_synthesis.md) — Multi-agent support (worktree + merge queue)
