# Gas Town: Architecture & Multi-Agent Model

**March 2026**

---

## 1. Overall System Architecture

Gas Town is a Go-based CLI (`gt`) that orchestrates multiple Claude Code instances across shared codebases using Git as its state machine and Beads as its task-tracking layer.

```
┌─────────────────────────────────────────────────┐
│                   TOWN (~/.gt)                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │  Mayor   │  │  Deacon  │  │   Dogs   │      │
│  │ (dispatch│  │ (health  │  │ (maint.) │      │
│  │  + human │  │  daemon) │  │          │      │
│  │  iface)  │  │          │  │          │      │
│  └────┬─────┘  └──────────┘  └──────────┘      │
│       │                                         │
│  ┌────┴────────────────────────────────────┐    │
│  │              RIG (per repo)             │    │
│  │  ┌─────────┐ ┌──────────┐ ┌─────────┐  │    │
│  │  │ Witness │ │ Refinery │ │  Beads   │  │    │
│  │  │ (super- │ │ (merge   │ │ (JSONL   │  │    │
│  │  │  visor) │ │  queue)  │ │  tasks)  │  │    │
│  │  └────┬────┘ └──────────┘ └─────────┘  │    │
│  │       │                                 │    │
│  │  ┌────┴──────────────────────────┐      │    │
│  │  │     AGENTS (per worktree)     │      │    │
│  │  │  ┌────────┐  ┌────────┐      │      │    │
│  │  │  │Polecat │  │Polecat │ ...  │      │    │
│  │  │  │(worker)│  │(worker)│      │      │    │
│  │  │  │wt: /a  │  │wt: /b  │      │      │    │
│  │  │  └────────┘  └────────┘      │      │    │
│  │  │  ┌────────┐                  │      │    │
│  │  │  │  Crew  │ (persistent)     │      │    │
│  │  │  └────────┘                  │      │    │
│  │  └───────────────────────────────┘      │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  Human Overseer (strategic decisions)           │
└─────────────────────────────────────────────────┘
```

Source: [GitHub README](https://github.com/steveyegge/gastown), [Torq Reading List](https://reading.torqsoftware.com/notes/software/ai-ml/agentic-coding/2026-01-15-gas-town-multi-agent-orchestration-framework/)

---

## 2. Agent Role Hierarchy

Gas Town defines seven agent roles across two levels:

### Town-Level (global coordination)
| Role | Purpose | Persistence |
|------|---------|-------------|
| **Mayor** | Chief dispatcher. Human-facing interface. Decomposes features, assigns work, monitors progress. | Persistent |
| **Deacon** | System health daemon. Monitors agent status, detects failures, triggers recovery. | Persistent |
| **Dogs** | Maintenance helpers. Cleanup tasks, queue management, housekeeping. | Ephemeral |

### Rig-Level (per-repository)
| Role | Purpose | Persistence |
|------|---------|-------------|
| **Crew** | Persistent named agents with long-running context about a specific project area. | Persistent |
| **Polecats** | Ephemeral worker agents. Spawn, execute a single Bead, commit, disappear. | Ephemeral |
| **Refinery** | Merge queue manager. Resolves conflicts, rebases branches, integrates completed work. | Persistent |
| **Witness** | Supervisor and unblocker. Monitors worker progress, nudges stalled agents, resolves blockers. | Persistent |

Source: [Maggie Appleton analysis](https://maggieappleton.com/gastown), [GitHub README](https://github.com/steveyegge/gastown)

---

## 3. Process Model: How Agents Are Spawned

Each agent runs as an independent Claude Code CLI process within a **dedicated git worktree**. This provides file-system isolation — agents cannot interfere with each other's working directory.

**Spawning flow:**
1. Mayor decomposes a high-level prompt into Beads (atomic tasks)
2. Mayor dispatches unblocked Beads to available Polecats via `gt sling` (hook attachment)
3. Each Polecat receives its Bead, spins up in an isolated worktree, executes the task
4. On completion, Polecat commits to its feature branch and signals completion
5. Refinery picks up completed branches and merges them into the integration branch

**Session management:** Gas Town runs agents in tmux sessions, enabling multiple parallel terminals. The `gt handoff` command enables session restarts while preserving context through Beads state.

Source: [Better Stack Guide](https://betterstack.com/community/guides/ai/gas-town-multi-agent/)

---

## 4. The MEOW Stack (Workflow Abstraction)

Gas Town layers workflow abstraction through five levels:

| Layer | Description | Analogy |
|-------|-------------|---------|
| **Beads** | Atomic tasks. JSONL stored in git. Persistent identity, status tracking. | GitHub Issues |
| **Epics** | Hierarchical collections organizing Beads into tree structures. | Epic/Story grouping |
| **Molecules** | Instantiated workflows — directed graphs with dependencies, gates, loops. Crash-recoverable. | CI pipeline run |
| **Protomolecules** | Reusable workflow templates. Define structure without binding to specific work. | CI pipeline definition |
| **Formulas** | TOML-based high-level specs "cooked" into protomolecules. | Makefile/Dockerfile |

The critical innovation is **Molecules**: durable workflow sequences where each step has acceptance criteria. If an agent crashes mid-step, the next session picks up from the last checkpoint. Gas Town calls this "nondeterministic idempotence" — the path varies but the outcome converges because the workflow definition and acceptance criteria persist.

Source: [Torq Reading List](https://reading.torqsoftware.com/notes/software/ai-ml/agentic-coding/2026-01-15-gas-town-multi-agent-orchestration-framework/)

---

## 5. Agent Communication

Agents do **not** communicate directly with each other. All coordination flows through:

1. **Beads** — shared task state in git. Agents read/write task status.
2. **Mail-as-Beads** — a messaging system with addressing modes (direct, list, queue, group) and priority levels. Messages are Beads that must be deleted after processing.
3. **Git state** — completed work is visible to other agents through committed code in worktrees.
4. **Witness nudges** — the supervisor periodically checks worker status and intervenes if agents stall.

There is no shared memory, no message bus, no real-time pub/sub. Git is the universal coordination layer.

Source: [Gas Town vs Swarm-Tools comparison](https://gist.github.com/johnlindquist/4174127de90e1734d58fce64c6b52b62)

---

## 6. Configuration Model

The operator defines the multi-agent topology through:

- **Rig registration**: `gt rig add <name> <repo-url>` — registers a repository
- **Crew definition**: `gt crew add <name> --rig <rig>` — creates persistent agent roles
- **Formula files**: TOML specs that define workflow templates for common operations (feature dev, bug fix, refactoring)
- **Role-specific CLAUDE.md files**: Each agent role receives customized system prompts defining its responsibilities and constraints

Source: [Better Stack Guide](https://betterstack.com/community/guides/ai/gas-town-multi-agent/)

---

## 7. Comparison with Rein Architecture

| Aspect | Gas Town | Rein |
|--------|----------|---------|
| **Dispatch model** | Concurrent (20-30 agents) | Sequential (one agent per task) |
| **Isolation** | Git worktrees | Subprocess per task |
| **Context monitoring** | None (crash and resume) | Real-time token tracking, zone-based intervention |
| **Task definition** | Mayor decomposition + Formula templates | Operator-defined task sequence |
| **Quality control** | Witness supervision (LLM-based) | Binary structured evaluation (deterministic) |
| **Cost tracking** | None (acknowledged gap) | Normalized per-task token accounting |
| **Agent selection** | Primarily Claude Code | Multi-agent support (Claude, Codex, Gemini) |

**Key insight:** Gas Town sidesteps context pressure entirely — agents crash and resume from Beads. Rein monitors and intervenes before degradation. These are complementary approaches: rein could serve as Gas Town's quality layer for individual sessions.
