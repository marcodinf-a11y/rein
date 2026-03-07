# Proposal: LEARNINGS.md Seed File

**Target:** TASKS.md (seed file pattern)
**Priority:** Now
**Effort:** Low

---

## Proposed Pattern: LEARNINGS.md for Multi-Session Workflows

The following describes a proposed seed file convention for operational knowledge transfer between sessions.

---

### LEARNINGS.md Seed File

In multi-session workflows, agents discover operational knowledge about the codebase — correct build commands, test runner quirks, compiler flags, naming conventions, file layout patterns. This knowledge is currently lost when the session ends and a fresh context starts.

LEARNINGS.md is an optional seed file that persists operational knowledge across sessions. The agent writes discoveries during execution; subsequent sessions receive LEARNINGS.md as a seed file and benefit from prior discoveries.

#### What Goes in LEARNINGS.md

Operational knowledge — facts about the codebase and environment that are useful for any session, not just the current task:

- Correct build/test commands (especially non-obvious ones)
- Compiler or linter quirks
- Test framework configuration
- File naming conventions
- Environment setup requirements
- Known issues or workarounds

#### What Does NOT Go in LEARNINGS.md

- Task-specific implementation details (those belong in code and commits)
- Conversation history or reasoning traces
- Large code snippets (those belong in seed files via `files`)

#### Agent Prompt Instruction

Add to task prompts for multi-session workflows:

```
If you discover useful facts about this codebase (build commands, test quirks,
naming conventions, environment requirements), append them to LEARNINGS.md.
Keep entries concise — one line per fact. Future sessions will read this file.
```

#### Example LEARNINGS.md

```markdown
# Learnings

- Build: `cargo build --release` (debug builds fail tests due to timing)
- Tests: `cargo test -- --test-threads=1` (DB tests are not parallelizable)
- Lint: `cargo clippy -- -D warnings` (CI enforces zero warnings)
- The `auth` module uses a custom JWT implementation, not the `jsonwebtoken` crate
- Config files are in `config/` not `src/config/` — the `Config` struct loads from there
- Python scripts in `scripts/` require Python 3.11+ (f-string syntax)
```

#### Lifecycle in Multi-Session Workflows

```
Session 1 (task: setup-db)
  ├─ Agent discovers: "cargo test -- --test-threads=1"
  ├─ Agent writes to LEARNINGS.md
  └─ Session ends

Session 2 (task: add-auth)
  ├─ LEARNINGS.md seeded from Session 1 output
  ├─ Agent reads learnings, uses correct test command
  ├─ Agent discovers: "auth module uses custom JWT"
  ├─ Agent appends to LEARNINGS.md
  └─ Session ends

Session 3 (task: add-rate-limiting)
  ├─ LEARNINGS.md seeded from Session 2 output
  ├─ Agent has all prior operational knowledge
  └─ ...
```

#### Task Definition Integration

For single-session tasks, LEARNINGS.md is just a seed file — set it in `files`:

```json
{
    "files": {
        "LEARNINGS.md": "# Learnings\n\n- Build: cargo build --release\n- Tests: cargo test -- --test-threads=1\n"
    }
}
```

For multi-session workflows, rein carries forward LEARNINGS.md from the prior session's sandbox into the next session's `files`. This is a workflow-level concern, not a task-level concern — rein reads LEARNINGS.md from the completed sandbox and injects it into the next task's seed files.

#### Size Management

LEARNINGS.md should remain concise — one line per fact, no prose. If it grows beyond ~50 lines, the operator should curate it between sessions. Unlike ralph-orchestrator's scratchpad (16K character cap with FIFO eviction), rein does not enforce an automatic size limit. The operator controls what persists.

---

## Rationale

Ralph-orchestrator's scratchpad (`scratchpad.rs`) serves a similar purpose: it persists information across iterations so the agent doesn't rediscover the same facts. Key differences:

| Dimension | Ralph scratchpad | LEARNINGS.md |
|-----------|-----------------|-------------|
| Format | Markdown with heading preservation | Markdown (one line per fact) |
| Size limit | 16K characters, FIFO eviction | No enforced limit (operator-curated) |
| Content | Anything the agent writes | Operational knowledge only |
| Persistence | Across iterations (same loop) | Across sessions (different loops) |
| Control | Automatic eviction | Operator curation |

Rein already has seed files (`files` in TaskDefinition) for carrying forward code artifacts. LEARNINGS.md is a formalized convention for a specific kind of seed content: **knowledge about the codebase**, not code itself. The distinction matters because:

1. **Knowledge compounds.** Each session adds facts. Code artifacts from session 1 may be irrelevant to session 3, but "the test runner needs `--test-threads=1`" is always useful.

2. **Knowledge is small.** A well-curated LEARNINGS.md is a few hundred tokens — negligible context cost with high value per token.

3. **Knowledge reduces wasted turns.** Without LEARNINGS.md, each session may spend 1–3 turns rediscovering build commands, test configurations, and environment quirks. With it, the agent starts productive immediately.

This is identified as "Now" priority in the [Ralph Wiggum synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) §7.3 and §8 because it requires no code changes — just a prompt convention and a workflow pattern for carrying the file forward.

## Source References

- [Ralph Wiggum Synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) — Section 7.3 (agent self-learning file)
- [Ralph Orchestrator Synthesis](../00_synthesis.md) — Section 3.4 (scratchpad budget management)
- [Ralph Orchestrator Architecture](../01_architecture.md) — scratchpad implementation details
- TASKS.md — `files` field (seed file mechanism)
