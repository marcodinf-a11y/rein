# Rein — Architecture

For the rationale behind key design decisions, see [Architecture Decision Records](docs/adr/).

## Glossary

- **Task** — a unit of work defined by a JSON (or YAML) file. One task = one prompt + optional files, setup commands, and validation commands.
- **Run** — a single invocation of `rein run`. A run may dispatch one task (file path) or many (directory scan).
- **Execution** — one task dispatched to one agent in one sandbox. A run with N tasks produces N executions.
- **Session** — the agent's conversation context within an execution. Sessions may be continued across executions via `--resume`/`--continue` (agent-dependent).

---

## System Overview

Rein is a Python CLI application that orchestrates AI coding agents with **context pressure monitoring** as its core function. It loads task definitions, creates isolated sandboxes, invokes agents as subprocesses, monitors their token consumption in real-time against the model's context window, and intervenes when pressure thresholds are crossed — ensuring work is persisted before quality degrades.

```
┌──────────────────────────────────────────────────────────┐
│                      rein CLI                             │
│                     (cli.py)                             │
├──────────────────────────────────────────────────────────┤
│                    Orchestrator                          │
│                    (runner.py)                           │
├──────────┬──────────┬─────────────┬──────────┬──────────┤
│ Dispatch │ Sandbox  │  Pressure   │ Capture  │ Evaluate │
│          │          │  Monitor    │          │          │
│ Load     │ Tempdir  │ Stream      │ Parse    │ Validate │
│ task     │ Worktree │ parse       │ JSON     │ Score    │
│ JSON     │ Copy     │ Track       │ Normalize│ Report   │
│ Invoke   │ Seed     │ tokens      │ tokens   │          │
│ agent    │ Setup    │ Zone check  │ Diff     │          │
│          │          │ Kill/wrap-up│ Artifacts│          │
├──────────┴──────────┴─────────────┴──────────┴──────────┤
│                  Agent Adapters                          │
│        claude.py    codex.py    gemini.py                │
│    (stream-json)   (JSONL)    (stream-json)              │
└──────────────────────────────────────────────────────────┘
```

## Project Structure

```
rein/
    pyproject.toml
    src/
        rein/
            __init__.py
            cli.py                  # Click-based CLI entrypoint
            runner.py               # Orchestrator: runs tasks against agents
            monitor.py              # Context pressure monitor (stream parser + zone logic)
            models.py               # Core dataclasses/TypedDicts
            tokens.py               # Token normalization + ContextPressure + ZoneConfig
            sandbox.py              # Execution isolation (worktrees/tmp dirs)
            learnings.py            # LEARNINGS.md injection, extraction, validation
            evaluate.py             # Result evaluation and scoring
            agents/
                __init__.py
                base.py             # AgentAdapter protocol
                claude.py           # Claude Code adapter
                codex.py            # Codex CLI adapter
                gemini.py           # Gemini CLI adapter
    tasks/
        example_fizzbuzz.json       # Example task definition
    results/                        # Default output directory (gitignored)
    tests/
```

## Five Subsystems

### 1. Task Dispatch (`runner.py`)

Loads a `TaskDefinition` from JSON and invokes the appropriate agent adapter. The orchestrator handles the full lifecycle: sandbox creation, agent invocation, output capture, validation, cleanup.

**Task validation:** JSON is validated against the schema at load time. If the file is syntactically invalid or missing required fields (`id`, `name`, `prompt`), Rein fails immediately with the parse/validation error before any sandbox or subprocess work.

**Specification validation:** After parsing, Rein runs two deterministic checks: prompt length (>= 50 characters) and validation commands presence (non-empty). Behavior depends on mode: `warn` (default) prints warnings and prompts the operator, `strict` blocks dispatch, `skip` disables checks. Configurable via `--spec-check` CLI flag or `[spec_validation]` in `rein.toml`. See [ADR-013](docs/adr/ADR-013-pre-dispatch-specification-validation.md).

**Pre-flight check:** Before creating a sandbox, the orchestrator calls `is_available()` on the resolved adapter. If the agent CLI is not found in `PATH`, Rein fails immediately with a clear error naming the missing binary and suggesting `rein list agents`. Authentication failures cannot be detected upfront (each agent authenticates differently) — they surface as subprocess errors with non-zero `exit_code`, `termination_reason=error`, and stderr captured in the report.

### 2. Execution Isolation (`sandbox.py`)

Creates an isolated sandbox for each run based on the task's `workspace` configuration (see [TASKS.md — Workspace Types](TASKS.md#workspace-types)):

| Type | Mechanism | Diff baseline | Cleanup |
|---|---|---|---|
| `tempdir` (default) | Empty temp directory | Initial commit after setup (if git init'd) | Delete temp directory |
| `worktree` | `git worktree add` from source repo | `HEAD` of source at creation time | `git worktree remove` + delete branch |
| `copy` | Copy source tree into temp directory | Current commit (git repo) or auto-created initial commit (non-git) | Delete temp directory |

After the sandbox is created, seed files from `files` are written (overwriting or adding), then `setup_commands` run. The agent's `cwd` is set to the sandbox directory.

#### LEARNINGS.md Injection and Extraction

Rein manages a persistent `LEARNINGS.md` file at `.rein/LEARNINGS.md` in the project root. This file carries operational knowledge (build commands, test quirks, naming conventions) across sessions.

**Injection (session start):** During sandbox setup, `learnings.inject()` copies `.rein/LEARNINGS.md` from the project root into the sandbox as `LEARNINGS.md`. If the project root file doesn't exist, an empty file with a `# Learnings` header is created. The agent is instructed to read this file before starting and append discoveries during execution (see [PROMPTS.md — Work Protocol](PROMPTS.md#3-work-protocol-defense-in-depth--fr-094)).

**Extraction (post-session):** After the quality gate's final verdict (pass, warn, or fail after exhausting `max_rounds`), `learnings.extract()` reads `LEARNINGS.md` from the sandbox, diffs it against the project root version, validates new entries structurally, and appends them to `.rein/LEARNINGS.md`. Extraction happens once — not after each retry round. See [ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md) for rationale.

**Structural validation (no LLM):** Each new entry must start with `- `, be ≤ 200 characters, and not duplicate an existing entry. If the file exceeds 80 lines after merge, rein warns the operator but does not truncate — the operator curates.

This is the first "write-back" pattern in rein. All other data flows are sandbox → report. The `inject`/`extract` interface in `learnings.py` is designed to be reusable for future write-back artifacts (e.g., DEFERRED.md).

#### Agent Git Identity

Every subprocess gets git author/committer environment variables so agent-authored commits are distinguishable from human commits in git history. Both `GIT_AUTHOR_*` and `GIT_COMMITTER_*` are set to the same values — no operator identity leaks into agent commits.

```
GIT_AUTHOR_NAME="{agent}/{model_short}"          # effort omitted when default
GIT_AUTHOR_NAME="{agent}/{model_short}/{effort}"  # effort included when explicit
GIT_AUTHOR_EMAIL="agent@rein.local"
GIT_COMMITTER_NAME="..."                          # same as AUTHOR
GIT_COMMITTER_EMAIL="agent@rein.local"
```

The name format uses short model aliases (`opus`, `sonnet`, `o3`, `flash`, `pro`) for readability. When effort is not specified (or resolves to the agent's default), the `/{effort}` segment is omitted entirely — do not emit `"default"` as a segment. See [ADR-001](docs/adr/ADR-001-agent-git-identity.md).

| Agent | Model | Effort | Git Author |
|-------|-------|--------|------------|
| Claude Code | opus | high | `claude-code/opus/high` |
| Claude Code | sonnet | *(default)* | `claude-code/sonnet` |
| Codex CLI | o3 | high | `codex-cli/o3/high` |
| Gemini CLI | pro | *(default)* | `gemini-cli/pro` |

The email is a fixed constant — not a real address. It satisfies git's email requirement and provides a single grep target for all rein-managed commits:

```bash
# All agent-authored commits
git log --author="agent@rein.local"

# Commits by a specific agent/model
git log --author="claude-code/opus"
```

These env vars are set on the agent subprocess and reused by rein when running wrap-up git commands (see [SESSIONS.md — Rein Wrap-Up Protocol](SESSIONS.md#rein-wrap-up-protocol)). This approach works for all three workspace types without requiring a `.git` directory at setup time — env vars take effect only when git commands actually run.

**Environment variable handling:** Adapters copy `os.environ` and delete known conflict variables before spawning the subprocess. This prevents the parent process's environment from interfering with agent CLIs. Known conflicts:

- **Claude:** `CLAUDECODE` — when set, `claude -p` refuses to run (detects nested invocation). The adapter must unset it.
- **Codex:** No known conflicts. `CODEX_API_KEY` is required and passed through.
- **Gemini:** No known conflicts.

If new agent CLIs introduce similar env var conflicts, add them to the adapter's scrub list and document them here.

Artifacts and diffs are captured *before* the sandbox is cleaned up.

#### Subprocess Termination Procedure

A shared procedure used whenever Rein needs to kill an agent subprocess — whether triggered by context pressure zone actions or wall-clock timeout. Two modes:

| Mode | When Used | Behavior |
|---|---|---|
| **Graceful** | Yellow zone (context pressure) | Wait for the agent's current turn to complete, then signal |
| **Immediate** | Red zone (context pressure), timeout | Signal immediately — do not wait for the current turn |

**Signal sequence:**

1. Send `SIGINT` for Codex (triggers `TurnAborted` event in JSONL), `SIGTERM` for Claude and Gemini
2. Wait up to 5 seconds for the process to exit
3. If still alive after 5 seconds, send `SIGKILL`

**Post-kill output recovery:**

- Drain the stdout buffer — all complete NDJSON/JSONL lines received before the kill point are captured and parseable
- Discard any incomplete final line (partial JSON is not recoverable)
- Since all three agents emit streaming NDJSON/JSONL output (per issue #6), partial output is always line-recoverable on kill — there is no truncated single-JSON blob problem

**Exit codes:** The process `returncode` is recorded as-is. Typical values: `-15` (SIGTERM), `-9` (SIGKILL), `130` (Codex SIGINT convention).

**Callers:** Context pressure zone actions ([SESSIONS.md — Zone Actions](SESSIONS.md#zone-actions)) and wall-clock timeout ([SESSIONS.md — Timeout](SESSIONS.md#timeout--wall-clock-limit-exceeded)).

### 3. Context Pressure Monitor (`monitor.py`)

The defining subsystem of Rein. Runs concurrently with the agent subprocess, reading the NDJSON/JSONL output stream line by line, computing context pressure after each turn, and triggering zone actions when thresholds are crossed.

- **Input:** Agent's stdout stream (NDJSON or JSONL), model context window size (from config or `modelUsage`), zone thresholds (from `ZoneConfig`)
- **Output:** `ContextPressure` result (zone, utilization, measurement method), kill signal if threshold crossed
- **Data structures:** `ContextPressure`, `ZoneConfig` — defined in [TOKENS.md](TOKENS.md)
- **Zone actions:** Green → continue; Yellow → graceful stop + Rein wrap-up; Red → immediate kill + Rein wrap-up
- **Degraded mode:** When mid-run tokens are unavailable (Gemini, Claude with extended thinking), the monitor reads the stream for tool/message events but can only compute pressure post-completion

See [SESSIONS.md — Context Pressure Monitoring](SESSIONS.md#context-pressure-monitoring) for the full protocol.

### 4. Output Capture (`agents/*.py`)

Each adapter parses the agent's stdout (JSON or JSONL), extracts the result text, and normalizes token usage into a unified model (see [TOKENS.md](TOKENS.md)). When the monitor has already parsed the stream (real-time mode), the capture step works from the buffered stream data rather than re-reading stdout. Raw output is preserved for debugging.

**Parse failure fallback:** If NDJSON/JSONL parsing fails (malformed output, unexpected format, encoding errors), Rein captures whatever raw stdout/stderr was received and stores it in the report as `raw_output`. Token accounting and result text extraction are unavailable — `normalized_tokens` is null and `result_text` is empty. The report sets `termination_reason=error` and `parse_error` contains the exception message. Sandbox artifacts (files, diffs) are still captured normally — parse failure only affects stream-derived fields.

### 5. Evaluation (`evaluate.py`)

Runs `validation_commands` in the sandbox after the agent finishes. Scoring is binary — all commands pass (score 1.0) or any fails (score 0.0). Results are written to a structured JSON report (see [REPORTS.md](REPORTS.md)).

**Completion promise:** Before running validation commands, the evaluator checks for a `.rein/complete` marker file in the sandbox. If present, the agent signaled that it believes the task is complete. The promise is cross-referenced against validation results to produce a four-outcome confidence classification: **confident**, **suspicious**, **overconfident**, or **incomplete**. See [ADR-002](docs/adr/ADR-002-completion-promise-signal.md) and [REPORTS.md](REPORTS.md) for the full matrix and report fields.

**Validation timeout:** Each validation command has a fixed 60-second timeout. If a command doesn't exit within 60 seconds, Rein kills it (SIGTERM → 5s grace → SIGKILL) and records it as failed.

## Agent Adapter Interface

Uses Python `Protocol` (structural typing). Adapters don't inherit from a base class — they match the shape:

```python
@runtime_checkable
class AgentAdapter(Protocol):
    """Protocol all agent adapters must satisfy."""

    @property
    def name(self) -> str:
        """Human-readable agent name, e.g. 'claude-code'."""
        ...

    def is_available(self) -> bool:
        """Check if the agent CLI is installed and callable."""
        ...

    async def run(
        self, task: TaskDefinition, sandbox_path: Path
    ) -> AgentResult:
        """Execute the task in the given sandbox directory."""
        ...
```

The `run` method is `async` because all three CLIs are subprocess invocations that benefit from `asyncio.create_subprocess_exec`.

## Token Normalization & Context Pressure Model

Token normalization — the `NormalizedTokenUsage` dataclass, normalization rules, field mapping table, `BudgetStatus` enum, and `analyze_budget` function — is documented in [TOKENS.md](TOKENS.md). Per-agent token field mappings and adapter code are in [AGENTS.md](AGENTS.md).

Context pressure data structures — `ContextPressure`, `ZoneConfig`, and the model context window lookup — are also defined in [TOKENS.md](TOKENS.md). The monitoring protocol and zone actions are in [SESSIONS.md — Context Pressure Monitoring](SESSIONS.md#context-pressure-monitoring).

## Execution Flow

1. Load task definition from JSON
2. Run specification validation checks (prompt length, validation commands presence). Mode controlled by `--spec-check` / `rein.toml`. See [ADR-013](docs/adr/ADR-013-pre-dispatch-specification-validation.md).
3. Resolve model context window (from config lookup or agent-reported metadata)
4. Create isolated sandbox per `workspace.type` (tempdir, worktree, or copy)
5. Seed files and run setup commands in sandbox
6. Invoke agent CLI as subprocess in sandbox directory (stream-json mode)
7. **Monitor stream in real-time** — parse NDJSON/JSONL line by line, compute context pressure per turn. Concurrently run wall-clock timer against `timeout_seconds`:
   - Green zone → continue reading stream
   - Yellow zone → [Subprocess Termination Procedure](#subprocess-termination-procedure) (graceful); `termination_reason=context_pressure`; proceed to Rein wrap-up (step 8a)
   - Red zone → [Subprocess Termination Procedure](#subprocess-termination-procedure) (immediate); `termination_reason=context_pressure`; proceed to Rein wrap-up (step 8a)
   - Degraded mode (no mid-run tokens) → read stream for events only, compute pressure post-completion
   - Timeout → if wall-clock exceeds `timeout_seconds`, [Subprocess Termination Procedure](#subprocess-termination-procedure) (immediate); `termination_reason=timed_out`; proceed to Rein wrap-up (step 8a)
8. Parse final or partial output from stream buffer; normalize tokens into unified model
   - 8a. *(If terminated by Rein — context pressure or timeout)* **Rein wrap-up:** commit uncommitted changes, write run log, update PROGRESS.md, log termination metrics. Optionally dispatch post-kill summary agent (yellow zone only — not dispatched for red zone or timeout).
9. Capture diff and artifacts before sandbox cleanup
10. Run validation commands in sandbox
11. Generate evaluation result (including `ContextPressure` in report)
12. **Extract learnings** — after final quality gate verdict, diff sandbox `LEARNINGS.md` against `.rein/LEARNINGS.md` in project root, validate new entries structurally, merge back. See [ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md).
13. Save structured JSON report to `results/{task_id}_{YYYYMMDD_HHMMSS}.json` (see [REPORTS.md](REPORTS.md))

## CLI Design

Built with [Click](https://click.palletsprojects.com/). Output formatting via [Rich](https://rich.readthedocs.io/).

```
Usage: rein [OPTIONS] COMMAND [ARGS]...

Commands:
  run       Run task(s) against agent(s)
  list      List available tasks or agents
  report    Generate summary from results
```

### `rein run`

```
Options:
  -t, --task PATH       Task JSON file or directory [required]
  -a, --agent TEXT      Agent: claude, codex, or gemini [default for tasks without agent field]
  -m, --model TEXT      Model: e.g. opus, sonnet, flash, pro [default for tasks without model field]
  --effort TEXT         Reasoning effort: low, medium, high [default for tasks without effort field]
  -o, --output PATH     Output directory [default: ./results]
  --budget INTEGER      Token budget override [default: 70000]
  --timeout INTEGER     Timeout override in seconds
  --spec-check TEXT     Spec validation mode: warn, strict, skip [default: warn]
  -y, --yes             Auto-confirm interactive prompts (spec warnings, etc.)
  --parallel            Dispatch tasks concurrently (future)
  --dry-run             Show what would execute without running
```

### `rein list agents`

```
claude-code    available    /home/user/.local/bin/claude (v2.1.63)
codex-cli      NOT FOUND    codex not in PATH
gemini-cli     available    /usr/bin/gemini (v0.16.0)
```

## Dependencies

| Package | Purpose |
|---|---|
| `click` | CLI framework |
| `rich` | Terminal output formatting |
| *(stdlib `asyncio`)* | Async runtime — subprocess orchestration via `asyncio.create_subprocess_exec` |
| *(stdlib `json`)* | Task definition parsing |

Python 3.12+. Linux, macOS, and Windows. No ML/AI libraries — Rein only *invokes* agents, it doesn't run models.

## Why Python

Rein is glue code — it launches subprocesses, parses JSON, writes reports. The choice of language matters less than in a performance-critical system, so the decision optimizes for development speed and ecosystem fit.

- **The workload is subprocess orchestration.** The agents are external CLIs. Rein calls them via `asyncio.create_subprocess_exec`, reads their stdout, and parses JSON. Python handles this fine — the agents are the bottleneck, not Rein.
- **`click` + `rich` is a proven CLI stack.** These are mature, well-documented libraries that get a polished CLI with minimal code. No equivalent pairing exists in Go or Rust with the same productivity.
- **Dataclasses and Protocols fit the domain.** `TaskDefinition`, `NormalizedTokenUsage`, and `AgentAdapter` map naturally to frozen dataclasses and structural typing. No framework or ORM needed.
- **Audience alignment.** Developers using AI coding agents are likely comfortable reading and extending Python. Lower barrier to contribution.
- **No runtime performance pressure.** Rein spends its time waiting for agents to finish. There is no hot loop, no high-throughput path, no latency-sensitive code.

Go or Rust would offer single-binary distribution and avoid the Python runtime dependency. If distribution becomes a pain point, that tradeoff can be revisited.

## Future: Multi-Agent

The architecture supports multi-agent workflow composition from day one:

- The `AgentAdapter` protocol is agent-agnostic
- `NormalizedTokenUsage` provides consistent token tracking regardless of agent
- Per-task `agent`, `model`, and `effort` fields enable composition pipelines
- The `--parallel` flag enables concurrent dispatch of different tasks

The current release implements single-agent sequential execution. See [ROADMAP.md](ROADMAP.md) for multi-agent plans.
