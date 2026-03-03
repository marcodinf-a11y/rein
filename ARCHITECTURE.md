# Agentic Harness — Architecture

## System Overview

The harness is a Python CLI application that orchestrates AI coding agents. It loads task definitions, creates isolated sandboxes, invokes agents as subprocesses, parses their output, normalizes token usage, and generates evaluation reports.

```
┌─────────────────────────────────────────────────┐
│                  harness CLI                     │
│                  (cli.py)                        │
├─────────────────────────────────────────────────┤
│               Orchestrator                       │
│               (runner.py)                        │
├──────────┬──────────┬───────────┬───────────────┤
│ Dispatch │ Sandbox  │  Capture  │  Evaluation   │
│          │          │           │               │
│ Load     │ Tempdir  │ Parse     │ Validate      │
│ task     │ Worktree │ JSON      │ Score         │
│ JSON     │ Copy     │ Normalize │ Report        │
│ Invoke   │ Seed     │ tokens    │               │
│ agent    │ Setup    │ Diff      │               │
├──────────┴──────────┴───────────┴───────────────┤
│              Agent Adapters                      │
│     claude.py   codex.py   gemini.py            │
└─────────────────────────────────────────────────┘
```

## Project Structure

```
agentic_harness/
    pyproject.toml
    src/
        agentic_harness/
            __init__.py
            cli.py                  # Click-based CLI entrypoint
            runner.py               # Orchestrator: runs tasks against agents
            models.py               # Core dataclasses/TypedDicts
            tokens.py               # Token normalization logic
            sandbox.py              # Execution isolation (worktrees/tmp dirs)
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

## Four Subsystems

### 1. Task Dispatch (`runner.py`)

Loads a `TaskDefinition` from JSON and invokes the appropriate agent adapter. The orchestrator handles the full lifecycle: sandbox creation, agent invocation, output capture, validation, cleanup.

### 2. Execution Isolation (`sandbox.py`)

Creates an isolated sandbox for each run based on the task's `workspace` configuration (see [TASKS.md — Workspace Types](TASKS.md#workspace-types)):

| Type | Mechanism | Diff baseline | Cleanup |
|---|---|---|---|
| `tempdir` (default) | Empty temp directory | Initial commit after setup (if git init'd) | Delete temp directory |
| `worktree` | `git worktree add` from source repo | `HEAD` of source at creation time | `git worktree remove` + delete branch |
| `copy` | Copy source tree into temp directory | Current commit (git repo) or auto-created initial commit (non-git) | Delete temp directory |

After the sandbox is created, seed files from `files` are written (overwriting or adding), then `setup_commands` run. The agent's `cwd` is set to the sandbox directory.

Artifacts and diffs are captured *before* the sandbox is cleaned up.

### 3. Output Capture (`agents/*.py`)

Each adapter parses the agent's stdout (JSON or JSONL), extracts the result text, and normalizes token usage into a unified model (see [TOKENS.md](TOKENS.md)). Raw output is preserved for debugging.

### 4. Evaluation (`evaluate.py`)

Runs `validation_commands` in the sandbox after the agent finishes. Scoring is binary — all commands pass (score 1.0) or any fails (score 0.0). Results are written to a structured JSON report (see [REPORTS.md](REPORTS.md)).

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

## Token Normalization Model

Token normalization — the `NormalizedTokenUsage` dataclass, normalization rules, field mapping table, `BudgetStatus` enum, and `analyze_budget` function — is documented in [TOKENS.md](TOKENS.md). Per-agent token field mappings and adapter code are in [AGENTS.md](AGENTS.md).

## Execution Flow

1. Load task definition from JSON
2. Create isolated sandbox per `workspace.type` (tempdir, worktree, or copy)
3. Seed files and run setup commands in sandbox
4. Invoke agent CLI as subprocess in sandbox directory
5. Parse agent's JSON output for token usage and result
6. Normalize tokens into unified model
7. Capture diff and artifacts before sandbox cleanup
8. Run validation commands in sandbox
9. Generate evaluation result
10. Save structured JSON report to `results/{task_id}_{YYYYMMDD_HHMMSS}.json` (see [REPORTS.md](REPORTS.md))

## CLI Design

Built with [Click](https://click.palletsprojects.com/). Output formatting via [Rich](https://rich.readthedocs.io/).

```
Usage: harness [OPTIONS] COMMAND [ARGS]...

Commands:
  run       Run task(s) against agent(s)
  list      List available tasks or agents
  report    Generate summary from results
```

### `harness run`

```
Options:
  -t, --task PATH       Task JSON file or directory [required]
  -a, --agent TEXT      Agent: claude, codex, gemini, or "all" [default: all]
  -o, --output PATH     Output directory [default: ./results]
  --budget INTEGER      Token budget override [default: 70000]
  --timeout INTEGER     Timeout override in seconds
  --parallel            Run agents in parallel (future)
  --dry-run             Show what would execute without running
```

### `harness list agents`

```
claude-code    available    /home/user/.local/bin/claude (v2.1.63)
codex-cli      NOT FOUND    codex not in PATH
gemini-cli     available    /usr/bin/gemini (v0.16.0)
```

## Dependencies

| Package | Purpose |
|---|---|
| `click` | CLI framework |
| *(stdlib `json`)* | Task definition parsing |
| `rich` | Terminal output formatting |

Python 3.12+. No ML/AI libraries — the harness only *invokes* agents, it doesn't run models.

## Why Python

The harness is glue code — it launches subprocesses, parses JSON, writes reports. The choice of language matters less than in a performance-critical system, so the decision optimizes for development speed and ecosystem fit.

- **The workload is subprocess orchestration.** The agents are external CLIs. The harness calls them via `asyncio.create_subprocess_exec`, reads their stdout, and parses JSON. Python handles this fine — the agents are the bottleneck, not the harness.
- **`click` + `rich` is a proven CLI stack.** These are mature, well-documented libraries that get a polished CLI with minimal code. No equivalent pairing exists in Go or Rust with the same productivity.
- **Dataclasses and Protocols fit the domain.** `TaskDefinition`, `NormalizedTokenUsage`, and `AgentAdapter` map naturally to frozen dataclasses and structural typing. No framework or ORM needed.
- **Audience alignment.** Developers using AI coding agents are likely comfortable reading and extending Python. Lower barrier to contribution.
- **No runtime performance pressure.** The harness spends its time waiting for agents to finish. There is no hot loop, no high-throughput path, no latency-sensitive code.

Go or Rust would offer single-binary distribution and avoid the Python runtime dependency. If distribution becomes a pain point, that tradeoff can be revisited.

## Future: Multi-Agent

The architecture supports multi-agent from day one:

- The `AgentAdapter` protocol is agent-agnostic
- `NormalizedTokenUsage` allows cross-agent comparison
- The `--agent all` flag and `--parallel` flag are designed for this

MVP implements single-agent sequential execution only. Multi-agent parallel dispatch is a documented future extension.
