# GSD Deep Dive: Planning Directory & State Management

**March 2026**

This document analyzes GSD's `.planning/` directory structure and STATE.md cross-session persistence model, comparing them with Rein's seed files and LEARNINGS.md pattern.

---

## 1. What GSD Installs

GSD is distributed via npm (`npx get-shit-done-cc`). The installer (`bin/install.js`, ~700 LOC JavaScript) copies markdown files into runtime-specific config directories:

- **Claude Code:** `~/.claude/commands/gsd/` (slash commands) + `~/.claude/get-shit-done/` (workflows, templates, references, agents)
- **OpenCode:** `~/.config/opencode/`
- **Gemini CLI:** `~/.gemini/`
- **Codex:** `~/.codex/skills/gsd-*/SKILL.md`

The installer also registers hooks in `settings.json` (statusline + context monitor) and copies agent definitions. It does not install a daemon, server, or background process. There is no GSD runtime — the "orchestration" happens within Claude Code's own agent spawning (Task tool) and slash command (Skill tool) mechanisms.

**Source:** `bin/install.js`, `package.json` (glittercowboy/get-shit-done, MIT, JavaScript, 25.5K stars, 2.2K forks, created Dec 14, 2025)

---

## 2. The `.planning/` Directory

GSD creates a `.planning/` directory in the project root during `/gsd:new-project`. This is the canonical state store for the entire project lifecycle.

### Structure

```
.planning/
    config.json              # Workflow settings, model profiles, gates
    STATE.md                 # Living memory — position, decisions, session continuity
    PROJECT.md               # Vision, constraints, key decisions table
    REQUIREMENTS.md          # Scoped v1/v2 requirements with phase traceability
    ROADMAP.md               # Phases mapped to requirements, progress tracking
    research/                # Domain research (stack, features, architecture, pitfalls)
        STACK.md
        FEATURES.md
        ARCHITECTURE.md
        PITFALLS.md
        SUMMARY.md
    phases/
        01-auth/
            01-CONTEXT.md    # User preferences from /gsd:discuss-phase
            01-RESEARCH.md   # Domain research for this phase
            01-1-PLAN.md     # Atomic task plan (XML structure)
            01-1-SUMMARY.md  # Execution results
            01-VERIFICATION.md  # Goal verification
            01-UAT.md        # User acceptance testing
        02-api/
            ...
    quick/                   # Ad-hoc task tracking
    todos/
        pending/
        done/
    debug/                   # Debug session state
        resolved/
```

**Source:** `get-shit-done/templates/`, `get-shit-done/workflows/new-project.md`, README.md

### What Each File Does

| File | Created By | Read By | Size Constraint |
|------|-----------|---------|-----------------|
| `config.json` | `/gsd:new-project` | Every workflow | ~40 lines |
| `STATE.md` | `/gsd:new-project` | First step of every workflow | < 100 lines |
| `PROJECT.md` | `/gsd:new-project` | Planning, execution | Full vision doc |
| `REQUIREMENTS.md` | `/gsd:new-project` | Planning, verification | Scoped v1/v2 |
| `ROADMAP.md` | `/gsd:new-project` | Navigation, progress | Phase list + progress |
| `CONTEXT.md` | `/gsd:discuss-phase` | Planner, researcher | Per-phase preferences |
| `PLAN.md` | `/gsd:plan-phase` | Executor | Atomic — fits in fresh context |
| `SUMMARY.md` | Executor agent | Progress, state updates | Per-plan results |

---

## 3. STATE.md: Cross-Session Persistence

STATE.md is GSD's most interesting architectural decision. It's a single markdown file (< 100 lines) that serves as "the project's short-term memory spanning all phases and sessions."

### Template Structure

```markdown
# Project State

## Project Reference
See: .planning/PROJECT.md (updated [date])
**Core value:** [One-liner]
**Current focus:** [Current phase name]

## Current Position
Phase: [X] of [Y] ([Phase name])
Plan: [A] of [B] in current phase
Status: [Ready to plan / Planning / Ready to execute / In progress / Phase complete]
Last activity: [YYYY-MM-DD] — [What happened]
Progress: [░░░░░░░░░░] 0%

## Performance Metrics
**Velocity:**
- Total plans completed: [N]
- Average duration: [X] min
- Total execution time: [X.X] hours

## Accumulated Context
### Decisions
[Recent decisions summary]

### Pending Todos
[Count + reference]

### Blockers/Concerns
[Active issues]

## Session Continuity
Last session: [YYYY-MM-DD HH:MM]
Stopped at: [Description]
Resume file: [Path or "None"]
```

**Source:** `get-shit-done/templates/state.md`

### STATE.md Lifecycle

1. **Created** after ROADMAP.md during `/gsd:new-project`
2. **Read first** in every workflow — progress, plan, execute, transition
3. **Updated** after every significant action via `gsd-tools.cjs` CLI commands:
   - `state advance-plan` — increments plan counter
   - `state update-progress` — recalculates progress bar from disk
   - `state record-metric` — appends execution metrics
   - `state add-decision` — adds to decisions section
   - `state record-session` — updates timestamp and stopped-at
4. **Size-constrained** to < 100 lines — "a DIGEST, not an archive"

The gsd-tools.cjs CLI (a Node.js helper script) handles atomic updates to STATE.md, preventing the common problem of LLMs corrupting structured markdown when doing read-modify-write cycles.

---

## 4. Comparison: GSD `.planning/` vs Rein Seed Files

| Dimension | GSD `.planning/` | Rein Seed Files |
|-----------|------------------|-------------------|
| **Persistence model** | Markdown files in project directory, committed to git | `files` dict in task JSON, written to sandbox at start |
| **Scope** | Entire project lifecycle (multi-phase, multi-session) | Single task execution |
| **Cross-session memory** | STATE.md (< 100 lines, updated programmatically) | LEARNINGS.md (proposed, not yet implemented) |
| **State management** | CLI tool (gsd-tools.cjs) manages STATE.md updates atomically | Rein captures reports, diffs, artifacts |
| **Size control** | Explicit constraints (STATE.md < 100 lines, plans fit in fresh context) | Token budget (70K default) sizes task scope |
| **Who writes it** | Agent writes (via CLI tool), rein reads | Operator writes task definition, agent writes code |
| **Recovery** | STATE.md + git history enables resume from any point | Task re-dispatch from scratch (fresh context) |

### Key Differences

**GSD is project-scoped, rein is task-scoped.** GSD tracks an entire project's lifecycle across phases and sessions. Rein tracks individual task executions. GSD's `.planning/` is more analogous to a project management system than to Rein's seed files.

**GSD relies on the agent to maintain state.** STATE.md is updated by the executor agent (via gsd-tools.cjs). If the agent crashes mid-update, STATE.md may be inconsistent. Rein captures state externally — the agent doesn't update rein state, rein observes the agent and records results.

**GSD's state drifts over time.** GitHub issue #956 documents systematic STATE.md drift after 18 phases: progress showed 100% when the milestone was 86% done, phase completion statuses mismatched between files, PROJECT.md was 4 phases stale. Root causes: gsd-tools.cjs regex bugs AND workflow gaps (PROJECT.md never updated in the standard cycle). The issue author noted: "Root cause is not agent error — agents follow the workflow specs correctly. The drift comes from three categories of gaps in the framework itself." This is a structural risk of agent-maintained state — Rein's external snapshot model avoids it.

**GSD's state is append-heavy, rein state is snapshot-based.** STATE.md accumulates decisions and metrics. Rein produces a fresh report per execution — each run is a self-contained record.

---

## 5. Comparison: STATE.md vs LEARNINGS.md

Rein's proposed LEARNINGS.md (from the Ralph Wiggum deep dive) and GSD's STATE.md solve overlapping problems with different designs.

| Dimension | GSD STATE.md | Rein LEARNINGS.md (proposed) |
|-----------|-------------|-------------------------------|
| **Purpose** | Project position + accumulated context + session continuity | Operational knowledge (build commands, quirks, patterns) |
| **Updated by** | Agent (via gsd-tools.cjs) | Agent (writes during execution) |
| **Size** | < 100 lines (enforced by template) | No constraint specified |
| **Content** | Decisions, blockers, progress, velocity | Build/test discoveries, compiler quirks, API patterns |
| **Scope** | Project-wide | Task-wide (persisted across sessions for same task) |
| **Persistence** | Committed to project git | Seeded into sandbox from prior run |

**STATE.md is broader but shallower.** It tracks project position and decisions but not operational knowledge. LEARNINGS.md is narrower but deeper — it carries forward specific technical discoveries that help future sessions avoid rediscovering the same information.

**Rein should have both.** A STATE.md-like file for project position (when multi-task mode exists) and LEARNINGS.md for operational knowledge (now, for single tasks that span multiple sessions).

---

## 6. The config.json Schema

GSD stores project settings in `.planning/config.json`:

```json
{
  "mode": "interactive",        // "yolo" | "interactive"
  "granularity": "standard",   // "coarse" | "standard" | "fine"
  "workflow": {
    "research": true,           // Spawn researchers before planning
    "plan_check": true,         // Verify plans before execution
    "verifier": true,           // Verify goals after execution
    "auto_advance": false,      // Auto-chain discuss → plan → execute
    "nyquist_validation": true  // Cross-reference validation
  },
  "planning": {
    "commit_docs": true,        // Git-track .planning/
    "search_gitignored": false
  },
  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "task_level": false,
    "skip_checkpoints": true,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2
  },
  "gates": {
    "confirm_project": true,
    "confirm_phases": true,
    "confirm_roadmap": true,
    "confirm_plan": true,
    "execute_next_plan": true,
    "issues_review": true,
    "confirm_transition": true
  },
  "safety": {
    "always_confirm_destructive": true,
    "always_confirm_external_services": true
  }
}
```

**Source:** `get-shit-done/templates/config.json`

This is notably more structured than most Claude Code frameworks. The gates system provides human-in-the-loop control at every workflow transition. The parallelization section controls wave execution granularity.

---

## 7. Implications for the Rein

### Adopt: STATE.md Size Constraint Pattern
The < 100-line constraint on STATE.md is well-motivated. For Rein's planned LEARNINGS.md, adopt a similar explicit size limit. Without it, agents will append indefinitely until the file itself becomes a context pressure source.

### Study: Programmatic State Updates
GSD's gsd-tools.cjs manages STATE.md updates atomically — the agent calls a CLI tool rather than editing the file directly. This prevents the common failure mode of LLMs corrupting structured markdown. If rein implements LEARNINGS.md, consider a structured format (JSONL?) rather than freeform markdown to avoid corruption.

### Skip: Full `.planning/` Structure
Rein operates at a different level of abstraction. GSD's `.planning/` is a project management system for human-directed development. Rein is an agent evaluation and monitoring framework. Adopting the full `.planning/` structure would be scope creep.

---

## Sources

- glittercowboy/get-shit-done (MIT, JavaScript, 25.5K stars, 2.2K forks)
- `get-shit-done/templates/state.md` — STATE.md template and lifecycle documentation
- `get-shit-done/templates/config.json` — config schema
- `get-shit-done/workflows/execute-phase.md` — state update protocol
- `agents/gsd-executor.md` — executor state management
- Rein docs: TOKENS.md, SESSIONS.md, TASKS.md
- Prior deep dive: ralph_wiggum_deep_dive/00_synthesis.md (LEARNINGS.md proposal)
