# Gas Town: Coordination & Conflict Resolution

**March 2026**

---

## 1. The Core Problem

Multiple agents editing the same codebase concurrently creates three classes of conflicts:

1. **File-level conflicts**: Two agents modify the same file simultaneously
2. **Semantic conflicts**: Two agents make independently valid but mutually incompatible changes (e.g., renaming a function one agent calls)
3. **Dependency conflicts**: Task A's output is Task B's input, but both run in parallel

Gas Town addresses all three, with varying degrees of success.

---

## 2. Worktree Isolation: Preventing File Conflicts

Gas Town's primary coordination mechanism is **git worktree isolation**. Each agent (Polecat/Crew) operates in its own git worktree — a separate working directory backed by the same repository. Agents never share a working directory.

**How it works:**
1. When a Polecat spawns, `gt` creates a new worktree branched from the integration branch
2. The agent works exclusively in its worktree, committing to a feature branch
3. On completion, the feature branch is submitted to the merge queue
4. The Refinery merges branches sequentially

This eliminates file-level conflicts during execution. Conflicts only surface at merge time, when the Refinery must reconcile divergent branches.

Source: [Gas Town vs Swarm-Tools comparison](https://gist.github.com/johnlindquist/4174127de90e1734d58fce64c6b52b62)

---

## 3. The Refinery: Merge Queue Management

The Refinery is Gas Town's dedicated merge agent. It handles the integration of completed work:

**Process:**
1. Completed feature branches enter a merge queue
2. Refinery processes them sequentially (not concurrently)
3. For each branch, Refinery attempts a rebase onto the integration branch
4. If rebase succeeds cleanly → merge
5. If conflicts arise → Refinery (an LLM agent) attempts to resolve them
6. For irreconcilable conflicts → Refinery may "creatively reimagine implementations"

**Critical observation:** The Refinery is an LLM resolving merge conflicts. This is powerful (it can understand semantic intent) but risky (it may resolve conflicts incorrectly). There is no structured verification that the Refinery's resolution is correct — it relies on the LLM's judgment.

**Bottleneck risk:** As agent count increases, the merge queue grows. Sequential merging becomes a bottleneck. Maggie Appleton identifies this as a key design bottleneck: "This offloads human merge responsibility but creates a bottleneck as changes accumulate."

Source: [Maggie Appleton analysis](https://maggieappleton.com/gastown), [Better Stack Guide](https://betterstack.com/community/guides/ai/gas-town-multi-agent/)

---

## 4. Dependency Management

Gas Town handles task dependencies through two mechanisms:

### 4.1 Molecule Dependencies
Molecules (workflow instances) define directed graphs with dependencies and gates. The Mayor dispatches only **unblocked** Beads — tasks whose dependencies are satisfied. Blocked tasks wait in the queue until their predecessors complete.

### 4.2 Convoy Ordering
Convoys group Beads into delivery units with explicit ordering. The Mayor respects ordering constraints when dispatching work.

**Limitation:** Dependency tracking is coarse-grained. Gas Town knows "Task B depends on Task A" but does not track fine-grained data dependencies (e.g., "Task B needs the function signature Task A will create"). If Task A changes its approach, Task B may proceed with stale assumptions.

Source: [Torq Reading List](https://reading.torqsoftware.com/notes/software/ai-ml/agentic-coding/2026-01-15-gas-town-multi-agent-orchestration-framework/)

---

## 5. What Happens When Two Agents Modify the Same File

Because of worktree isolation, two agents **can** modify the same file simultaneously — they're working on different copies. The conflict surfaces at merge time:

1. Agent A completes, merges its branch via Refinery
2. Agent B completes, submits its branch to the merge queue
3. Refinery attempts rebase of B onto the now-updated integration branch
4. Git reports a merge conflict on the shared file
5. Refinery (LLM) reads both versions and attempts resolution
6. If successful, merge proceeds. If not, the Bead is flagged for human intervention.

**No file-level locking exists.** Gas Town explicitly avoids locking (unlike Swarm-Tools, which uses a reservation/claim system). The philosophy is: let agents work freely, resolve conflicts at merge time.

**Trade-off:** This maximizes parallelism but risks wasted work. If two agents make conflicting changes to the same core file, one agent's work may be partially or fully discarded during merge resolution.

Source: [Gas Town vs Swarm-Tools comparison](https://gist.github.com/johnlindquist/4174127de90e1734d58fce64c6b52b62)

---

## 6. Comparison: How Others Solve This

| System | Isolation | Conflict Resolution | Dependency Tracking |
|--------|-----------|--------------------|--------------------|
| **Gas Town** | Git worktrees | LLM-based Refinery merge | Molecule DAGs |
| **Stripe Minions** | Isolated agent scopes (read-only views) | Blueprint/minion pattern avoids conflicts (minions don't merge) | Blueprint defines scope |
| **Goose/Goosetown** | Subagent isolation | Town Wall (append-only log) + orchestrator review | Research/build/review phases |
| **Swarm-Tools** | File pattern reservations + DurableLock | Compare-and-swap locking | Event sourcing + checkpoints |
| **Ralph Orchestrator** | Worktree per loop + merge queue | Sequential merge (similar to Refinery) | Inter-iteration backpressure gates |
| **Rein (planned)** | Subprocess per task (sequential) | N/A (no concurrent execution) | Operator-defined task order |

**Key insight:** Stripe Minions avoids the problem entirely by making agents read-only (minions propose changes, humans review and apply). Gas Town tackles it head-on with LLM-based merge resolution. Rein's planned sequential execution avoids it by not running agents concurrently.

Source: `research/stripe_deep_dive/00_synthesis.md`, `research/ralph_orchestrator_deep_dive/00_synthesis.md`

---

## 7. Implications for Rein

### Sequential is sufficient for now
Gas Town's experience shows that merge management is the hardest unsolved problem in concurrent multi-agent coding. Rein's planned sequential multi-task approach (different agent per task, one at a time) avoids this entirely. This is a feature, not a limitation.

### When concurrent execution becomes necessary
If rein eventually supports concurrent agents, it should:
1. **Use worktree isolation** (proven in both Gas Town and ralph-orchestrator)
2. **Implement a merge queue** with structured verification (not just LLM judgment)
3. **Partition work by directory/module** to minimize cross-agent file conflicts
4. **Track fine-grained dependencies** beyond task-level ordering

### What to avoid
- LLM-only merge resolution without verification (Gas Town's Refinery has no quality gate)
- No-locking philosophy at scale (works for 5 agents, becomes chaotic at 30)
- Coarse dependency tracking (Gas Town's biggest coordination weakness)
