# Rein — Quality Gate & Pipeline Configuration

How rein evaluates agent output beyond pass/fail, manages retry rounds, and integrates review agents. Configuration lives in `rein.toml` at the project root.

---

## Design Inspiration: Stripe Minions

The quality gate and pipeline design is closely modeled on Stripe's **Minions** system — the most proven production-scale agentic CI/CD pipeline as of early 2026 (1,000+ merged PRs/week, zero human-written code). Key patterns adopted from Stripe:

| Stripe Pattern | Rein Equivalent |
|----------------|-------------------|
| **Blueprint pattern**: deterministic nodes (checkout, lint, test) + open-ended agent loops (coding) | Quality gate signals are the deterministic nodes; the agent run is the open-ended loop |
| **Max 2 CI rounds per task** | Configurable `max_rounds` (default: 2) with structured feedback between rounds |
| **Pre-warmed devboxes** (10s spin-up) | Sandbox isolation (worktree/tempdir/copy) with setup commands |
| **Pull requests as approval gates** | Quality gate verdict controls acceptance; review agent provides the approval signal |
| **Isolated from production/internet** | Sandbox with configurable network and filesystem restrictions |
| **Invoked via Slack** | Invoked via CLI, git hooks, or any trigger (rein is the execution engine, not the trigger) |

Where rein diverges from Stripe:

| Difference | Stripe | Rein |
|------------|--------|---------|
| **Scale** | Enterprise (thousands of engineers, dedicated infra team) | Solo/small-team (local-first, no infra dependency) |
| **Agent** | Forked Block Goose agent + "Toolshed" MCP server (400+ tools) | Any CLI agent (Claude Code, Codex, Gemini) via adapter protocol |
| **Trigger** | Slack bot | CLI, git hooks, or external trigger — rein is agnostic |
| **Context pressure** | Not documented | Core metric — real-time monitoring with zone-based intervention |
| **Tool ecosystem** | Custom MCP server with 400+ internal tools | Standard MCP; tools are the agent's responsibility, not rein's |

The Stripe model validates the core architectural bet: deterministic quality checks wrapped around an agent's creative work, with a hard cap on retry rounds to prevent token waste. Rein adapts this for individual developers and small teams without requiring enterprise infrastructure.

Sources: [Stripe Minions Part 1](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents), [Stripe Minions Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)

---

## Overview

The quality gate aggregates multiple signals into a verdict before accepting agent output. Each signal checks one dimension of quality — tests, lint, diff scope, context pressure, code review. Signals are configurable per-project in `rein.toml`.

The gate operates after each round of agent work. If the verdict is `"fail"` and rounds remain, rein dispatches a retry with feedback. If the verdict is `"pass"` or `"warn"`, or rounds are exhausted, the pipeline stops.

---

## rein.toml

### Location and Discovery

Rein looks for `rein.toml` in the following order:

1. Path specified via `--config` CLI flag
2. Current working directory
3. Git repository root (if inside a git repo)

If no `rein.toml` is found, rein uses built-in defaults.

### Reference from CLAUDE.md

```markdown
## Rein Configuration

This project uses the rein for quality-gated agent workflows.
See `rein.toml` for quality gate signals, context pressure thresholds, and defaults.
```

Any agent reading CLAUDE.md knows where to find the config. The toml itself is agent-agnostic.

### Resolution Order

```
rein.toml defaults  <  task definition fields  <  CLI flags
(lowest priority)                                    (highest priority)
```

`rein.toml` sets project defaults. Task JSON can override per-task. CLI flags override everything. Same pattern as the existing agent/model/effort resolution in [TASKS.md](TASKS.md).

### Full Schema

```toml
# rein.toml — project-level configuration for the rein

[project]
name = "my-project"

# ── Defaults ────────────────────────────────────────
# These apply to all tasks unless overridden per-task or per-CLI.

[defaults]
agent = "claude"
model = "sonnet"
effort = "high"
token_budget = 70000
timeout_seconds = 300
max_rounds = 2

# ── Specification Validation ──────────────────────
# Pre-dispatch checks on task definitions. See ADR-013.
# CLI --spec-check flag overrides mode.

[spec_validation]
mode = "warn"                      # "warn" | "strict" | "skip"
min_prompt_length = 50             # minimum prompt character count
require_validation_commands = true # require non-empty validation_commands

# ── Context Pressure ───────────────────────────────
# Global zone boundaries. See TOKENS.md for the ContextPressure model.

[context_pressure]
green_max_pct = 60
yellow_max_pct = 80

# Per-model overrides for models with large windows but early degradation.
[context_pressure.overrides."claude-opus-4-6"]
green_max_pct = 40
yellow_max_pct = 60

# ── Quality Gate ────────────────────────────────────
# Each [quality_gate.<name>] section defines a signal.
# Built-in signal names have special behavior (see Signal Types below).
# Any other name is treated as a custom command-based signal.

[quality_gate.build]
enabled = false                    # Enable for compiled languages (C, C++, C#)
required = true
commands = []

[quality_gate.tests]
enabled = true
required = true
commands = []                      # Empty = use task's validation_commands

[quality_gate.lint]
enabled = true
required = true
commands = []                      # Set per-project (see Language Examples below)

[quality_gate.typecheck]
enabled = false
required = false
commands = []

[quality_gate.diff_size]
enabled = true
required = false
max_files_changed = 20
max_lines_added = 500
max_lines_removed = 200
exclude_patterns = []

[quality_gate.context_pressure]
enabled = true
required = true
max_zone = "green"

[quality_gate.review]
enabled = false                    # Disabled by default — opt-in
required = true
agent = "claude"
model = "sonnet"
effort = "high"
token_budget = 30000
timeout_seconds = 180
include_task_spec = true
include_diff = true
include_validation_output = true

# ── Escalation Report ─────────────────────────────
# Generated when final_verdict == "fail". Always includes a mechanical
# summary; opt-in LLM enrichment via summary_agent. See ADR-012.

[escalation]
summary_agent = false              # Opt-in LLM-enriched summaries
summary_model = "haiku"            # Cheap/fast model for summary
summary_token_budget = 10000
summary_timeout_seconds = 60
```

---

## Signal Types

### Built-in Signals

Built-in signals have special behavior rein understands natively. They cannot be redefined as custom signals.

| Signal | Special Behavior | Default State |
|--------|-----------------|---------------|
| `build` | Blocks `tests` on failure (dependency) | Disabled |
| `tests` | Falls back to task's `validation_commands` when `commands = []` | Enabled, required |
| `lint` | Command-based; no fallback | Enabled, required |
| `typecheck` | Command-based; no fallback | Disabled |
| `diff_size` | Parses `git diff --stat`; has `max_*` fields and `exclude_patterns` | Enabled, optional |
| `context_pressure` | Reads from monitor data; no commands to run | Enabled, required |
| `review` | Dispatches a second agent; expects structured JSON output | Disabled |

### Custom Signals

Any `[quality_gate.<name>]` section where `<name>` is not a built-in signal name is a custom command-based signal. Custom signals follow the same shape as built-in command signals:

```toml
[quality_gate.<name>]
enabled = true
required = true | false
commands = ["command1", "command2"]
after = "build"                    # Optional: ordering dependency (signal name or list)
```

Custom signals run commands in the sandbox, check exit codes (all zero = pass, any non-zero = fail), and capture stdout/stderr. No special parsing — rein treats them as opaque pass/fail checks.

---

## Signal Execution Order

Signals run in dependency order. Built-in signals have implicit ordering. Custom signals specify ordering via the `after` field.

```
build ──> tests ──> lint ──> typecheck ──> diff_size ──> context_pressure
  |         |                                                    |
  |         +── skipped if build fails                           v
  |                                                      custom signals
  +── skipped if disabled                                (ordered by `after`)
                                                                 |
                                                                 v
                                                              review
                                                          (always last)
```

### Ordering Rules

1. `build` runs first (if enabled)
2. `tests` runs after `build` (skipped if `build` fails)
3. `lint`, `typecheck`, `diff_size`, `context_pressure` run after `tests` (independently — they do not depend on each other)
4. Custom signals run after all built-ins, ordered by their `after` field
5. `review` runs last — it benefits from all other signal outputs

### Dependency Skipping

When a signal fails and a downstream signal depends on it, the downstream signal is skipped with `status = "skip"`. A skipped required signal does **not** cause a gate failure — only the root failure counts.

```python
# Example: build fails
# tests → skipped (depends on build)
# lint → runs (independent of build)
# Gate fails because build is required and failed, not because tests were skipped.
```

### The `after` Field

Custom signals default to running after all built-ins. The `after` field overrides this:

```toml
[quality_gate.sanitizers]
enabled = true
required = false
commands = ["cmake --build build-asan/", "./build-asan/tests"]
after = "build"

[quality_gate.integration_tests]
enabled = true
required = false
commands = ["./run_integration.sh"]
after = "tests"

[quality_gate.e2e]
enabled = true
required = false
commands = ["npx playwright test"]
after = ["build", "tests"]      # Multiple dependencies
```

Circular dependencies are a configuration error — rein rejects them at load time.

---

## Signal Result

Every signal produces a `SignalResult`:

```python
@dataclass(frozen=True)
class SignalResult:
    """Result of a single quality gate signal evaluation."""
    name: str                    # Signal name (e.g. "tests", "lint", "security")
    status: str                  # "pass" | "fail" | "warn" | "skip"
    required: bool               # From rein.toml
    message: str = ""            # Human-readable summary
    details: dict = field(default_factory=dict)  # Signal-specific data
```

### Status Values

| Status | Meaning |
|--------|---------|
| `pass` | Signal check succeeded |
| `fail` | Signal check failed |
| `warn` | Signal check produced warnings but did not fail |
| `skip` | Signal was skipped (dependency failed or signal disabled) |

### Per-Signal Details

**build:**
```json
{"commands_run": 1, "exit_code": 0, "output": "Build succeeded", "duration_seconds": 12.3}
```

**tests:**
```json
{"commands_run": 3, "commands_passed": 3, "output": "15 passed in 2.1s"}
```

**lint:**
```json
{"tool": "ruff", "issues": 0, "output": "All checks passed!"}
```

**typecheck:**
```json
{"tool": "mypy", "errors": 2, "output": "src/foo.py:12: error: ..."}
```

**diff_size:**
```json
{
    "files_changed": 5,
    "lines_added": 120,
    "lines_removed": 30,
    "exceeds_max_files": false,
    "exceeds_max_added": false,
    "exceeds_max_removed": false,
    "excluded_files": ["package-lock.json"]
}
```

**context_pressure:**
```json
{
    "final_zone": "green",
    "utilization_pct": 45.2,
    "measurement": "realtime",
    "model_context_window": 200000,
    "estimated_tokens_used": 90400
}
```

**review:**
```json
{
    "verdict": "approve",
    "issues": [
        {"file": "src/parser.py", "line": 42, "severity": "warning",
         "message": "Consider adding bounds check"}
    ],
    "summary": "Clean implementation. One minor suggestion.",
    "review_tokens": {"input": 8500, "output": 1200}
}
```

**custom signal:**
```json
{"commands_run": 1, "exit_code": 0, "output": "...", "duration_seconds": 3.1}
```

---

## Verdict Logic

The quality gate aggregates all signal results into a single verdict:

```python
def compute_verdict(signals: list[SignalResult]) -> str:
    """Compute gate verdict from signal results.

    - Any required signal fails → "fail"
    - No required failures but any signal warns or optional fails → "warn"
    - All signals pass or skip → "pass"
    """
    if any(s.status == "fail" and s.required for s in signals):
        return "fail"
    if any(s.status in ("fail", "warn") for s in signals):
        return "warn"
    return "pass"
```

```python
@dataclass(frozen=True)
class QualityGateResult:
    """Aggregate quality gate result for one round."""
    verdict: str                  # "pass" | "fail" | "warn"
    signals: list[SignalResult]
    round_number: int
    max_rounds: int
```

---

## Review Agent

The review signal dispatches a separate agent to review the implementation agent's output. It runs in a fresh context with its own token budget and timeout.

### Input

The review agent receives a structured prompt built from three sources:

| Input | Controlled By | Purpose |
|-------|--------------|---------|
| Original task spec | `include_task_spec = true` | What was asked |
| Git diff | `include_diff = true` | What was done |
| Validation output | `include_validation_output = true` | What passed/failed |

### Prompt Template

```
You are a code reviewer. Review the following changes against the original specification.

## Task Specification
{task_prompt}

## Changes (git diff)
{diff}

## Test/Lint Results
{validation_output}

## Other Signal Results
{signal_summaries}

## Instructions
Respond with JSON only:
{
    "verdict": "approve" or "request_changes",
    "issues": [{"file": "...", "line": 0, "severity": "error|warning|info", "message": "..."}],
    "summary": "one paragraph assessment"
}
```

### Review Agent Constraints

- **Read-only sandbox** — the review agent cannot modify code
- **Separate token budget** — defaults to 30,000; does not eat into the implementation budget
- **Context pressure monitored** — if the review agent itself reaches yellow/red, the review is unreliable and marked as `"warn"`
- **Structured output** — the agent must return parseable JSON; parse failure → `status = "warn"` with raw output in details

### Review Verdict Values

| Verdict | Gate Effect |
|---------|-------------|
| `approve` | Signal passes |
| `request_changes` | Signal fails (issues fed back to next round) |

---

## Round Mechanism

When the quality gate verdict is `"fail"` and `round_number < max_rounds`, rein dispatches a retry round.

### Round Flow

```
Round 1: Agent implements in sandbox
         → Quality gate evaluates
         → Verdict: fail
         → round < max_rounds? Yes → continue to round 2

Round 2: Same sandbox (code from round 1 is already there)
         Fresh agent context (no leftover pressure)
         Prompt includes:
           - Original task spec
           - Quality gate feedback (what failed and why)
         → Quality gate evaluates
         → Verdict: pass/fail/warn
         → round < max_rounds? No → stop, report final verdict

If max_rounds exhausted with verdict "fail": generate escalation report (see below).
```

### Round 2+ Prompt

Rein auto-generates a feedback prompt for retry rounds:

```
Your previous attempt had the following issues:

## Quality Gate Feedback
- build: PASS
- tests: FAIL — 2 of 5 tests failed (test_edge_case, test_empty_input)
- lint: PASS
- review: request_changes — "Missing null check in parse_input()"

## Test Output
{validation_output}

## Review Issues
{review_issues}

Fix these issues. The code is already in the working directory from your previous attempt.
```

### Round Constraints

- Each round gets a **fresh agent context** (zero context pressure)
- The **sandbox persists** across rounds — code from previous rounds is present
- Each round has its own **token budget** and **timeout** (from rein.toml defaults or task definition)
- The review agent runs fresh in each round — it does not carry context from previous reviews
- `max_rounds` is configurable (default: 2). Setting `max_rounds = 1` disables retries.
- **Learnings extraction happens once** after the final verdict (pass, warn, or fail), not after each round. This ensures only the final sandbox state — which reflects the cumulative work of all rounds — is persisted to `.rein/LEARNINGS.md`. See [ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md).

---

## Escalation Report

When the final verdict is `"fail"` (all rounds exhausted), rein generates a structured escalation report — a self-contained failure narrative for the developer picking up the work. See [ADR-012](docs/adr/ADR-012-structured-escalation-report.md).

The escalation report is always generated mechanically (zero cost, zero failure modes). If `[escalation] summary_agent = true` in `rein.toml`, a summary agent enriches the `approach_summary` and `diagnostic_summary` fields with semantic context. On summary agent failure, the mechanical version is preserved.

### What the escalation report contains

| Field | Description |
|-------|-------------|
| `task_summary` | Task name — quick identifier |
| `rounds_attempted` / `max_rounds` | How many rounds ran vs. the limit |
| `round_history[]` | Per-round narrative: what failed, what passed, approach summary |
| `escalation_trigger` | Why escalation happened: `max_rounds_exhausted`, `context_pressure`, `timed_out`, or `error` |
| `completion_confidence` | From final round's completion promise ([ADR-002](docs/adr/ADR-002-completion-promise-signal.md)): `overconfident` or `incomplete` |
| `learnings_snapshot` | Operational facts the agent discovered during execution ([ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)) |
| `diagnostic_summary` | Overall assessment with progress classification (improved/stagnated/regressed) |
| `preserved_state` | Where to find the agent's work: branch, commit SHA, diff stat |
| `summary_agent_used` | Whether LLM enrichment was used |

### Progress classification

Rein compares failing signals across rounds to classify progress:
- **improved**: final round has fewer failing signals than round 1
- **stagnated**: same number of failing signals
- **regressed**: more failing signals than round 1

### `output_excerpt` extraction

Each failing signal includes an `output_excerpt` — enough context to be actionable without duplicating full output:
1. Scan for failure markers (`FAILED`, `FAIL:`, `Error:`, `AssertionError`, etc.)
2. If found: up to 30 lines from the first marker
3. If not found: last 30 lines of output
4. Full output remains in `rounds[].evaluation.validation_output`

### Implementation module

`escalation.py` provides:
- `build_report(rounds, task, config) -> dict`
- `mechanical_summary(rounds) -> str`
- `extract_output_excerpt(validation_output, max_lines=30) -> str | None`
- `classify_progress(rounds) -> str`

---

## Report Format (Extended)

The quality gate extends the existing report format from [REPORTS.md](REPORTS.md). The flat `results[]` and `evaluations[]` arrays are replaced by a `rounds[]` array, where each round bundles its agent result, evaluation, and quality gate outcome.

### Extended Report Schema

```json
{
    "task": {
        "id": "refactor-auth-001",
        "name": "Refactor auth module to use JWT",
        "prompt": "...",
        "token_budget": 70000
    },
    "config": {
        "rein_toml": "rein.toml",
        "max_rounds": 2,
        "quality_gate_enabled": true,
        "signals_enabled": ["build", "tests", "lint", "context_pressure", "review"]
    },
    "rounds": [
        {
            "round_number": 1,
            "result": {
                "agent_name": "claude-code",
                "model": "sonnet",
                "effort": "high",
                "task_id": "refactor-auth-001",
                "exit_code": 0,
                "termination_reason": "completed",
                "normalized_tokens": {
                    "input_tokens": 45000,
                    "output_tokens": 12000,
                    "cache_read_tokens": 30000,
                    "cache_write_tokens": 8000,
                    "total_tokens": 57000
                },
                "budget_status": "within",
                "budget_analysis": {
                    "total_tokens": 57000,
                    "budget": 70000,
                    "utilization_pct": 81.4,
                    "status": "warning",
                    "remaining": 13000,
                    "cache_efficiency": {
                        "cache_read_tokens": 30000,
                        "cache_write_tokens": 8000
                    }
                },
                "duration_seconds": 42.1,
                "cost_usd": 0.065,
                "result_text": "...",
                "diff": "diff --git a/src/auth.py ...",
                "artifacts": {},
                "timestamp": "2026-03-06T14:20:00"
            },
            "evaluation": {
                "agent_name": "claude-code",
                "task_id": "refactor-auth-001",
                "validation_passed": false,
                "validation_output": "FAILED test_edge_case ...",
                "score": 0.0,
                "notes": ""
            },
            "quality_gate": {
                "verdict": "fail",
                "signals": [
                    {
                        "name": "tests",
                        "status": "fail",
                        "required": true,
                        "message": "2 of 5 tests failed",
                        "details": {"commands_run": 5, "commands_passed": 3}
                    },
                    {
                        "name": "lint",
                        "status": "pass",
                        "required": true,
                        "message": "No issues",
                        "details": {"tool": "ruff", "issues": 0}
                    },
                    {
                        "name": "diff_size",
                        "status": "pass",
                        "required": false,
                        "message": "5 files, +120/-30 lines",
                        "details": {
                            "files_changed": 5,
                            "lines_added": 120,
                            "lines_removed": 30,
                            "exceeds_max_files": false,
                            "exceeds_max_added": false,
                            "exceeds_max_removed": false,
                            "excluded_files": []
                        }
                    },
                    {
                        "name": "context_pressure",
                        "status": "pass",
                        "required": true,
                        "message": "Green zone (45.2%)",
                        "details": {
                            "final_zone": "green",
                            "utilization_pct": 45.2,
                            "measurement": "realtime",
                            "model_context_window": 200000
                        }
                    },
                    {
                        "name": "review",
                        "status": "fail",
                        "required": true,
                        "message": "Requested changes: missing null check",
                        "details": {
                            "verdict": "request_changes",
                            "issues": [
                                {
                                    "file": "src/auth.py",
                                    "line": 42,
                                    "severity": "error",
                                    "message": "Missing null check in parse_input()"
                                }
                            ],
                            "summary": "Implementation mostly correct but missing edge case handling.",
                            "review_tokens": {"input": 8500, "output": 1200}
                        }
                    }
                ]
            }
        },
        {
            "round_number": 2,
            "result": {
                "agent_name": "claude-code",
                "model": "sonnet",
                "effort": "high",
                "task_id": "refactor-auth-001",
                "exit_code": 0,
                "termination_reason": "completed",
                "normalized_tokens": {
                    "input_tokens": 38000,
                    "output_tokens": 8000,
                    "cache_read_tokens": 25000,
                    "cache_write_tokens": 6000,
                    "total_tokens": 46000
                },
                "budget_status": "within",
                "budget_analysis": {
                    "total_tokens": 46000,
                    "budget": 70000,
                    "utilization_pct": 65.7,
                    "status": "within",
                    "remaining": 24000,
                    "cache_efficiency": {
                        "cache_read_tokens": 25000,
                        "cache_write_tokens": 6000
                    }
                },
                "duration_seconds": 31.5,
                "cost_usd": 0.048,
                "result_text": "...",
                "diff": "diff --git a/src/auth.py ...",
                "artifacts": {},
                "timestamp": "2026-03-06T14:21:15"
            },
            "evaluation": {
                "agent_name": "claude-code",
                "task_id": "refactor-auth-001",
                "validation_passed": true,
                "validation_output": "5 passed in 1.2s",
                "score": 1.0,
                "notes": ""
            },
            "quality_gate": {
                "verdict": "pass",
                "signals": [
                    {
                        "name": "tests",
                        "status": "pass",
                        "required": true,
                        "message": "5 of 5 tests passed",
                        "details": {"commands_run": 5, "commands_passed": 5}
                    },
                    {
                        "name": "lint",
                        "status": "pass",
                        "required": true,
                        "message": "No issues",
                        "details": {"tool": "ruff", "issues": 0}
                    },
                    {
                        "name": "diff_size",
                        "status": "pass",
                        "required": false,
                        "message": "5 files, +125/-30 lines",
                        "details": {
                            "files_changed": 5,
                            "lines_added": 125,
                            "lines_removed": 30
                        }
                    },
                    {
                        "name": "context_pressure",
                        "status": "pass",
                        "required": true,
                        "message": "Green zone (38.1%)",
                        "details": {
                            "final_zone": "green",
                            "utilization_pct": 38.1
                        }
                    },
                    {
                        "name": "review",
                        "status": "pass",
                        "required": true,
                        "message": "Approved",
                        "details": {
                            "verdict": "approve",
                            "issues": [],
                            "summary": "Edge case fixed correctly. All tests passing.",
                            "review_tokens": {"input": 9000, "output": 800}
                        }
                    }
                ]
            }
        }
    ],
    "learnings": {
        "extracted": true,
        "new_entries": [
            "- test: pytest tests/test_auth.py requires --timeout=60 for JWT validation tests"
        ],
        "total_lines": 8,
        "warnings": []
    },
    "escalation_report": null,
    "final_verdict": "pass",
    "total_rounds": 2,
    "total_tokens": {
        "implementation": {
            "input_tokens": 83000,
            "output_tokens": 20000,
            "total_tokens": 103000
        },
        "review": {
            "input_tokens": 17500,
            "output_tokens": 2000,
            "total_tokens": 19500
        },
        "combined": {
            "input_tokens": 100500,
            "output_tokens": 22000,
            "total_tokens": 122500
        }
    },
    "total_cost_usd": 0.113,
    "total_duration_seconds": 95.3,
    "timestamp": "2026-03-06T14:22:00"
}
```

### Key Report Fields

| Field | Description |
|-------|-------------|
| `config` | Records which rein.toml was used and key pipeline settings |
| `rounds[]` | Each round bundles its agent result, evaluation, and quality gate verdict |
| `rounds[].quality_gate` | Per-round aggregate verdict and signal breakdown |
| `learnings` | Extraction results: new entries added, total line count, warnings ([ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)) |
| `escalation_report` | Structured failure narrative when `final_verdict == "fail"`, otherwise `null` ([ADR-012](docs/adr/ADR-012-structured-escalation-report.md)) |
| `final_verdict` | The verdict from the last round — the one that matters |
| `total_rounds` | How many rounds were needed (1 = first attempt passed) |
| `total_tokens` | Aggregate across all rounds, split by implementation vs review |
| `total_cost_usd` | Sum across all rounds and review agents |
| `total_duration_seconds` | Wall clock for the full pipeline including all rounds |

### Backward Compatibility

For runs without a quality gate (`quality_gate_enabled = false` or no rein.toml found):

- `rounds` has one entry
- `quality_gate` contains only `tests` and `context_pressure` signals (minimum built-in checks)
- `final_verdict` reflects the single round
- The report remains valid and parseable by tools expecting the extended format

---

## diff_size Signal — Exclude Patterns

The `exclude_patterns` field filters files from the diff size calculation. This prevents false positives from generated files, lock files, and build artifacts.

```toml
[quality_gate.diff_size]
enabled = true
required = false
max_files_changed = 20
max_lines_added = 500
max_lines_removed = 200
exclude_patterns = [
    "package-lock.json",
    "yarn.lock",
    "pnpm-lock.yaml",
    "*.csproj",
    "*.sln",
    "*.pbxproj",
    "Cargo.lock",
    "go.sum",
    "poetry.lock",
    "Pipfile.lock",
]
```

Implementation uses git pathspec excludes:

```bash
git diff --stat -- . ':!package-lock.json' ':!yarn.lock' ...
```

Patterns follow gitignore syntax. Both filename matches and glob patterns are supported.

---

## Language Examples

### Python

```toml
[quality_gate.build]
enabled = false

[quality_gate.tests]
enabled = true
required = true
commands = []                      # Uses task's validation_commands

[quality_gate.lint]
enabled = true
required = true
commands = ["ruff check ."]

[quality_gate.typecheck]
enabled = true
required = false
commands = ["mypy src/"]

[quality_gate.diff_size]
enabled = true
required = false
exclude_patterns = ["poetry.lock", "Pipfile.lock"]
```

### JavaScript / TypeScript

```toml
[quality_gate.build]
enabled = true                     # TypeScript needs compilation
required = true
commands = ["npm run build"]

[quality_gate.tests]
enabled = true
required = true
commands = ["npm test"]

[quality_gate.lint]
enabled = true
required = true
commands = ["npx eslint ."]

[quality_gate.typecheck]
enabled = false                    # tsc already runs in build

[quality_gate.diff_size]
enabled = true
required = false
exclude_patterns = ["package-lock.json", "yarn.lock", "pnpm-lock.yaml"]
```

### C / C++

```toml
[quality_gate.build]
enabled = true
required = true
commands = ["cmake --build build/"]

[quality_gate.tests]
enabled = true
required = true
commands = ["ctest --test-dir build/ --output-on-failure"]

[quality_gate.lint]
enabled = true
required = false
commands = ["clang-tidy src/*.cpp -- -std=c++20 -Iinclude/"]

[quality_gate.typecheck]
enabled = false                    # Compiler handles this

[quality_gate.diff_size]
enabled = true
required = false
exclude_patterns = ["CMakeFiles/", "*.cmake", "CMakeCache.txt"]

# Custom: memory safety check
[quality_gate.sanitizers]
enabled = true
required = false
commands = ["cmake --build build-asan/", "./build-asan/run_tests"]
after = "build"
```

### C#

```toml
[quality_gate.build]
enabled = true
required = true
commands = ["dotnet build --no-restore"]

[quality_gate.tests]
enabled = true
required = true
commands = ["dotnet test --no-build"]

[quality_gate.lint]
enabled = true
required = false
commands = ["dotnet format --verify-no-changes"]

[quality_gate.typecheck]
enabled = false                    # Compiler handles this

[quality_gate.diff_size]
enabled = true
required = false
exclude_patterns = ["*.csproj", "*.sln", "obj/", "bin/"]
```

---

## Custom Signal Examples

Custom signals extend the quality gate with project-specific checks. Any `[quality_gate.<name>]` section where `<name>` is not a built-in signal name is a custom signal.

```toml
# Security audit (JavaScript)
[quality_gate.security]
enabled = true
required = true
commands = ["npm audit --audit-level=high"]

# License compliance
[quality_gate.license]
enabled = true
required = false
commands = ["license-checker --failOn 'GPL-3.0'"]

# Documentation coverage (Python)
[quality_gate.docs]
enabled = false
required = false
commands = ["interrogate -v src/ --fail-under 80"]

# Bundle size (JavaScript)
[quality_gate.bundle_size]
enabled = true
required = false
commands = ["npx size-limit"]
after = "build"

# Memory safety (C/C++)
[quality_gate.sanitizers]
enabled = true
required = false
commands = ["cmake --build build-asan/", "./build-asan/run_tests"]
after = "build"

# Integration tests (any language)
[quality_gate.integration]
enabled = true
required = false
commands = ["./scripts/run_integration_tests.sh"]
after = "tests"

# End-to-end tests (JavaScript)
[quality_gate.e2e]
enabled = false
required = false
commands = ["npx playwright test"]
after = ["build", "tests"]
```

---

## Generic Signal Protocol

All signals — built-in and custom — share a common protocol:

```python
@dataclass(frozen=True)
class SignalConfig:
    """Configuration for a quality gate signal."""
    name: str
    enabled: bool = True
    required: bool = False
    commands: list[str] = field(default_factory=list)
    after: str | list[str] | None = None
```

Built-in signals extend this with their specific fields:

| Signal | Extra Fields |
|--------|-------------|
| `diff_size` | `max_files_changed`, `max_lines_added`, `max_lines_removed`, `exclude_patterns` |
| `context_pressure` | `max_zone` |
| `review` | `agent`, `model`, `effort`, `token_budget`, `timeout_seconds`, `include_task_spec`, `include_diff`, `include_validation_output` |

Custom signals use `SignalConfig` as-is. Adding a new check to a project requires only a toml section — no code changes.

---

## Pipeline Modes

Rein exposes three pipeline modes, all governed by the same quality gate:

```bash
# Forward: agent implements from task spec, quality gate evaluates
rein run -t task.json

# Review only: quality gate on existing changes (no implementation agent)
rein review --diff HEAD~1..HEAD

# Composition: sequential tasks from a directory, each independently gated
rein run -t tasks/pipeline/
```

### `rein review`

Review-only mode skips the implementation agent and runs the quality gate against existing changes. This is the git-hook entry point:

```bash
# .git/hooks/pre-push
rein review --diff HEAD~1..HEAD --config rein.toml
```

In review mode:
- `build`, `tests`, `lint`, `typecheck` run against the current working tree
- `diff_size` measures the specified diff range
- `context_pressure` is not applicable (no agent ran) — signal is skipped
- `review` dispatches the review agent against the diff
- No rounds — review mode is single-pass

---

## Interaction with Existing Specs

| Spec | Relationship |
|------|-------------|
| [TOKENS.md](TOKENS.md) | `context_pressure` signal reads from the same `ContextPressure` model. `ZoneConfig` in rein.toml replaces the standalone YAML config example. |
| [SESSIONS.md](SESSIONS.md) | Zone actions (graceful stop, immediate kill) remain unchanged. The quality gate runs **after** the agent finishes or is stopped. |
| [REPORTS.md](REPORTS.md) | The extended report format supersedes the flat `results[]`/`evaluations[]` structure. Single-round runs without quality gate remain compatible. `escalation_report` field added for terminal failures ([ADR-012](docs/adr/ADR-012-structured-escalation-report.md)). |
| [TASKS.md](TASKS.md) | Task definitions are unchanged. `validation_commands` feeds the `tests` signal and is checked by pre-dispatch spec validation ([ADR-013](docs/adr/ADR-013-pre-dispatch-specification-validation.md)). `token_budget` and `timeout_seconds` apply per-round. |
| [AGENTS.md](AGENTS.md) | Agent adapters are unchanged. The review agent is invoked through the same adapter interface. |
| [ARCHITECTURE.md](ARCHITECTURE.md) | The quality gate is a new subsystem between Output Capture and the final Report step. The execution flow gains a round loop around steps 6-12. Learnings extraction ([ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)) runs after the final verdict. |
