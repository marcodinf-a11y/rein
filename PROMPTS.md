# Rein — Prompt Assembly

How Rein constructs the final prompt delivered to agents. Covers the assembly pipeline, injected instructions, and per-agent delivery mechanisms.

---

## Design Principles

1. **The task author writes the task prompt. Rein wraps it.** The `prompt` field in the task definition is the core instruction. Rein adds operational instructions around it — never modifies it.
2. **Injected instructions are invisible to the task author.** They don't need to write "commit frequently" in every task — Rein handles that.
3. **Minimal token overhead.** The wrapper adds ~300–400 tokens. For a 70k budget, that's <0.6%.
4. **Agent-agnostic assembly, agent-specific delivery.** The assembled prompt is the same regardless of agent. How it reaches the agent differs per CLI.

---

## Assembly Pipeline

```
TaskDefinition.prompt          (human-written)
        │
        ▼
┌─────────────────────┐
│  Prompt Assembler   │
│  (prompts.py)       │
│                     │
│  1. Preamble        │  ← Rein identity + task metadata
│  2. Task prompt     │  ← TaskDefinition.prompt (verbatim)
│  3. Work protocol   │  ← Defense-in-depth instructions
│  4. Deviation rules │  ← What the agent must NOT do
│  5. Completion       │  ← How to signal "done"
└─────────────────────┘
        │
        ▼
  Assembled prompt (single string)
        │
        ▼
  Agent adapter delivers via stdin
```

The assembler is a pure function: `assemble_prompt(task: TaskDefinition, config: AssemblyConfig) -> str`. No side effects, no I/O. Easily testable.

---

## Prompt Sections

### 1. Preamble

Brief context so the agent knows it's running under Rein. This helps the agent interpret sandbox artifacts (LEARNINGS.md, PROGRESS.md) and understand why `.rein/complete` matters.

```
You are working on a task managed by Rein, a context-pressure-aware orchestrator.

Task: {task.name}
Task ID: {task.id}
```

**Why include this?** Without it, agents may be confused by LEARNINGS.md or PROGRESS.md appearing in the sandbox. The preamble sets expectations.

### 2. Task Prompt

The `TaskDefinition.prompt` field, inserted verbatim. No modification, no wrapping, no escaping beyond what's needed for the delivery mechanism.

```
{task.prompt}
```

### 3. Work Protocol (Defense in Depth — FR-094)

Operational instructions that ensure work is persisted incrementally. These protect against mid-run kills by ensuring the wrap-up protocol only needs to capture the last uncommitted delta.

```
## Work Protocol

- Commit after each meaningful change with a descriptive message. Do not batch all changes into a single final commit. Stage files individually — never use `git add .` or `git add -A`.
- After each commit, update PROGRESS.md with what was completed and what remains.
- If LEARNINGS.md exists, read it before starting. If you discover reusable insights (gotchas, patterns, constraints), append them — one line per fact, keep the file under 80 lines. Prefix entries with a category: `build:`, `test:`, `lint:`, `env:`, `convention:`, `quirk:`.
- If you encounter pre-existing issues unrelated to this task, log them in DEFERRED.md (one line per issue: file path, description) but do not fix them.
```

**Why these rules?** Each instruction protects against a specific failure mode observed in practice:
- **Commit frequently + stage individually:** Ensures the wrap-up protocol only captures the last delta, and prevents accidental commits of generated files, secrets, or sandbox artifacts (adapted from GSD's commit protocol: "NEVER `git add .`").
- **Update PROGRESS.md:** Creates a human-readable record of progress that survives agent termination.
- **LEARNINGS.md:** Carries operational knowledge across sessions. Capped at 80 lines to prevent unbounded growth (validated by GSD's STATE.md size constraint). Category prefixes (`build:`, `test:`, etc.) make the file scannable for operators and future sessions. Each entry must be a single line ≤ 200 characters — rein validates this structurally during post-session extraction (see [ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)).
- **DEFERRED.md:** Prevents scope creep while preserving signal. Without an explicit log target, agents either fix pre-existing issues (scope creep) or silently ignore them (lost signal). Adapted from GSD's `deferred-items.md` pattern.

### 4. Deviation Rules (FR-095)

Constraints that prevent the agent from going off-track. These are especially important for brownfield tasks where the agent has access to a full codebase and might refactor unrelated code.

The rules are adapted from GSD's 4-rule deviation framework (see [GSD Deep Dive — Deviation Rules](research/gsd_deep_dive/04_spec_driven_development.md#4-deviation-rules)), restructured for Rein's external orchestration model.

```
## Constraints

Automatic fixes (no permission needed):
- Fix bugs in code you wrote or modified for this task.
- Add missing error handling, validation, or imports needed to make your changes work.
- Install dependencies required by this task.

Scope boundaries:
- Do not modify files outside the scope of this task unless required to make the code compile or tests pass.
- Do not fix pre-existing issues unrelated to this task. Log them in DEFERRED.md instead.
- Do not delete or rename existing tests unless the task explicitly asks for it.

Guard rails:
- If a fix attempt fails 3 times, log the issue in DEFERRED.md and move on.
- If you are unsure whether a change is in scope, err on the side of not making it.
```

**Why this structure?** GSD's research showed that agents need explicit permission for "obvious" fixes (bugs, missing imports) — without it, they either fix too aggressively (refactoring unrelated code) or too conservatively (leaving their own broken code unfixed). The 3-tier structure (automatic / boundary / guard rail) gives clear guidance without being overly restrictive.

**Per-task deviation rules:** The task definition can include additional constraints via a new optional `constraints` field (list of strings). If present, these are appended after the default rules:

```python
@dataclass(frozen=True)
class TaskDefinition:
    # ... existing fields ...
    constraints: list[str] = field(default_factory=list)  # Additional deviation rules
```

### 5. Completion Signal (FR-090)

Instructions for the agent to verify its work and signal when the task is complete. The self-check step is adapted from GSD's executor protocol, which requires agents to verify their own claims before declaring success.

```
## Completion

Before signaling completion, verify your work:
1. Confirm that files you created or modified exist and are non-empty.
2. Run any tests or checks relevant to your changes.
3. Review PROGRESS.md — does it accurately reflect what was done?

When verified, create the file .rein/complete to signal completion.
```

Rein cross-references this marker with validation results:
- `.rein/complete` exists + validation passes → high confidence completion
- `.rein/complete` exists + validation fails → agent thought it was done but wasn't (interesting signal for evaluation)
- `.rein/complete` missing + validation passes → agent didn't signal but work is correct
- `.rein/complete` missing + validation fails → incomplete or failed

---

## Full Assembled Prompt (Template)

```
You are working on a task managed by Rein, a context-pressure-aware orchestrator.

Task: {task.name}
Task ID: {task.id}

---

{task.prompt}

---

## Work Protocol

- Commit after each meaningful change with a descriptive message. Do not batch all changes into a single final commit. Stage files individually — never use `git add .` or `git add -A`.
- After each commit, update PROGRESS.md with what was completed and what remains.
- If LEARNINGS.md exists, read it before starting. If you discover reusable insights (gotchas, patterns, constraints), append them — one line per fact, keep the file under 80 lines. Prefix entries with a category: `build:`, `test:`, `lint:`, `env:`, `convention:`, `quirk:`.
- If you encounter pre-existing issues unrelated to this task, log them in DEFERRED.md (one line per issue: file path, description) but do not fix them.

## Constraints

Automatic fixes (no permission needed):
- Fix bugs in code you wrote or modified for this task.
- Add missing error handling, validation, or imports needed to make your changes work.
- Install dependencies required by this task.

Scope boundaries:
- Do not modify files outside the scope of this task unless required to make the code compile or tests pass.
- Do not fix pre-existing issues unrelated to this task. Log them in DEFERRED.md instead.
- Do not delete or rename existing tests unless the task explicitly asks for it.

Guard rails:
- If a fix attempt fails 3 times, log the issue in DEFERRED.md and move on.
- If you are unsure whether a change is in scope, err on the side of not making it.
{extra_constraints}

## Completion

Before signaling completion, verify your work:
1. Confirm that files you created or modified exist and are non-empty.
2. Run any tests or checks relevant to your changes.
3. Review PROGRESS.md — does it accurately reflect what was done?

When verified, create the file .rein/complete to signal completion.
```

Where `{extra_constraints}` is the task's `constraints` field, each prefixed with `- `.

---

## Per-Agent Delivery

The assembled prompt is a single string. How it reaches the agent depends on the CLI:

| Agent | Delivery | Mechanism |
|---|---|---|
| **Claude Code** | stdin | Pipe assembled prompt to `claude -p --output-format stream-json --dangerously-skip-permissions` via `asyncio.create_subprocess_exec(stdin=PIPE)`. The `-p` flag reads from stdin. |
| **Codex CLI** | stdin | Pipe assembled prompt to `codex exec --json --full-auto --skip-git-repo-check` via stdin. |
| **Gemini CLI** | stdin | Pipe assembled prompt to `gemini --output-format stream-json --yolo` via stdin. |

All three agents accept prompts via stdin. This is the standard delivery method — it avoids `ARG_MAX` limits on Linux for long prompts and is consistent across agents.

### Why Not `--append-system-prompt`?

Claude Code supports `--append-system-prompt` to inject instructions into the system prompt rather than the user message. Reasons to use stdin instead:

1. **Consistency.** All three agents use the same delivery path.
2. **Visibility.** The agent sees the full context as a single coherent message. System prompt instructions can be deprioritized by the model.
3. **Testability.** The assembled prompt is a single string that can be logged, diffed, and replayed.
4. **No flag divergence.** Codex and Gemini don't have equivalent flags. Using `--append-system-prompt` for Claude only would mean different prompt structures per agent.

**Exception:** If testing shows that certain instructions (deviation rules, work protocol) are more reliably followed as system prompt instructions, the Claude adapter can split delivery — task prompt via stdin, Rein instructions via `--append-system-prompt`. This is an optimization, not the default.

---

## Assembly Configuration

```python
@dataclass(frozen=True)
class AssemblyConfig:
    """Controls which sections are included in the assembled prompt."""
    include_preamble: bool = True
    include_work_protocol: bool = True
    include_deviation_rules: bool = True
    include_completion_signal: bool = True
```

All sections are on by default. Configuration exists for:
- **Debugging:** Disable Rein instructions to see how the agent performs with the raw task prompt.
- **Benchmarking:** Compare agent performance with and without defense-in-depth instructions.
- **Minimal mode:** For very short tasks where the overhead matters.

Configurable via project config:

```yaml
prompt_assembly:
  include_preamble: true
  include_work_protocol: true
  include_deviation_rules: true
  include_completion_signal: true
```

---

## Module: `prompts.py`

```python
def assemble_prompt(task: TaskDefinition, config: AssemblyConfig | None = None) -> str:
    """Assemble the final prompt from task definition and Rein instructions.

    Pure function. No I/O, no side effects.
    """
```

This is a low-complexity module — string concatenation with conditionals. It should be implemented and tested early (alongside `models.py`) because every integration test depends on it.

---

## Sandbox Seeding (Interaction with `sandbox.py`)

The prompt references files that Rein seeds into the sandbox:

| File | Seeded by | Contents | Referenced in prompt section |
|---|---|---|---|
| `LEARNINGS.md` | `learnings.py` (FR-091) | Copied from `.rein/LEARNINGS.md` in project root, or empty with `# Learnings` header. Extracted back after final verdict ([ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)). | Work Protocol |
| `PROGRESS.md` | `sandbox.py` | Empty, agent writes to it | Work Protocol |
| `DEFERRED.md` | `sandbox.py` | Empty, agent writes to it | Work Protocol, Constraints |
| `.rein/` | `sandbox.py` | Directory created for completion signal | Completion |

These files are seeded *in addition to* the task's `files` field. The task author doesn't need to include them — Rein handles it.

---

## Token Cost Estimate

Approximate token counts for each section (measured with `tiktoken` o200k_base):

| Section | Tokens |
|---|---|
| Preamble | ~40 |
| Work Protocol | ~110 |
| Deviation Rules (default) | ~150 |
| Completion Signal | ~60 |
| Separators and formatting | ~15 |
| **Total Rein overhead** | **~375** |

With per-task constraints adding ~15 tokens each. Against a 70k budget, this is ~0.5% overhead.

---

## Open Questions

1. **Should the preamble include the timeout?** Telling the agent "you have 300 seconds" could cause it to rush. Omitting it means the agent can't self-manage pace. Current decision: omit — Rein manages the clock, not the agent.

2. **Should the work protocol mention context pressure?** The original SESSIONS.md suggests "context pressure awareness in agent prompt." But telling an agent "you might be killed at 60% context usage" may cause anxiety-driven shortcuts. Current decision: omit explicit pressure mention — the commit-frequently instruction achieves the same goal without the side effect.

3. **Per-agent prompt variations?** Currently the assembled prompt is agent-agnostic. If agents respond differently to instruction formatting (e.g., Claude follows markdown headers better, Codex prefers flat text), per-agent formatting could be added as an adapter concern. Current decision: start uniform, diverge only with evidence.
