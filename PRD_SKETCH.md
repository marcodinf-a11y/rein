# Rein — Product Requirements Document

Sources: BRIEF.md, ARCHITECTURE.md, TASKS.md, TOKENS.md, SESSIONS.md, AGENTS.md, REPORTS.md, ROADMAP.md.

---

## 1. Vision & Problem Statement

Rein is a context-pressure-aware orchestrator for AI coding agents. It monitors how agents consume their context windows during real development work, intervenes when quality is at risk, and ensures work is persisted before degradation sets in.

Its purpose is context-pressure-aware orchestration during real development work. Structured, normalized reporting makes cross-agent comparison a natural capability — but comparison is a byproduct, not the goal.

**The five pillars:** Task Dispatch, Execution Isolation, Context Pressure Monitoring, Output Capture, Evaluation.

See [BRIEF.md](BRIEF.md) for full problem statement and positioning.

---

## 2. Functional Requirements

### Task Loading & Dispatch

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-001 | Load task definition from JSON file, validate against schema | TASKS.md | `models.py` |
| FR-002 | Fail immediately on syntactically invalid JSON or missing required fields (`id`, `name`, `prompt`) | ARCHITECTURE.md | `models.py` |
| FR-003 | Load all `.json` files when given a directory path | TASKS.md | `runner.py` |
| FR-004 | Resolve agent from task `agent` field or CLI `--agent` flag; error if neither provides one | TASKS.md | `runner.py` |
| FR-005 | Resolve model and effort from task fields or CLI flags; task-level wins over CLI | TASKS.md | `runner.py` |
| FR-006 | Pre-flight check: call `is_available()` on adapter before sandbox creation; fail with clear error if agent CLI not in PATH | ARCHITECTURE.md | `runner.py` |

### Execution Isolation

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-010 | Create tempdir sandbox (empty temp directory) | ARCHITECTURE.md | `sandbox.py` |
| FR-011 | Create worktree sandbox (`git worktree add` from source repo, detached branch `rein/<task_id>`) | ARCHITECTURE.md | `sandbox.py` |
| FR-011a | Default source to cwd: if `workspace.type` is `worktree` or `copy` and neither CLI `--source` nor task `workspace.source` is provided, default to cwd. Fail if cwd is not a git repo for worktree type | TASKS.md | `sandbox.py` |
| FR-012 | Create copy sandbox (copy source tree into temp directory, `git init` if non-git source) | ARCHITECTURE.md | `sandbox.py` |
| FR-013 | Seed files from task `files` field into sandbox (create directories as needed) | TASKS.md | `sandbox.py` |
| FR-014 | Run `setup_commands` in sandbox after seeding | TASKS.md | `sandbox.py` |
| FR-015 | Capture diff and artifacts before sandbox cleanup | ARCHITECTURE.md | `sandbox.py` |
| FR-015a | Record baseline commit SHA at sandbox creation (before agent invocation). Tempdir: `git init && git add -A && git commit` after seeding/setup. Worktree/copy: capture `HEAD` SHA at creation time. Store as `baseline_sha` in run context | ARCHITECTURE.md | `sandbox.py` |
| FR-015b | After agent finishes (before cleanup): `git diff <baseline_sha> HEAD` (full patch) and `git diff --stat <baseline_sha> HEAD` (summary). Store as `diff_patch` and `diff_stat` in report. Write patch to `results/{task_id}_{timestamp}.patch` | ARCHITECTURE.md | `sandbox.py`, `runner.py` |
| FR-016 | Clean up sandbox after run (delete tempdir/copy, `git worktree remove` + branch delete for worktree) | ARCHITECTURE.md | `sandbox.py` |

### Agent Invocation

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-020 | Invoke agent CLI as async subprocess (`asyncio.create_subprocess_exec`) with `cwd` set to sandbox | ARCHITECTURE.md | `agents/*.py` |
| FR-021 | Set agent git identity (`GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`) in subprocess environment | ROADMAP.md | `agents/base.py` |
| FR-022 | Claude adapter: invoke with `--output-format stream-json`, `-p`, `--dangerously-skip-permissions`; unset `CLAUDECODE` env var | AGENTS.md | `agents/claude.py` |
| FR-023 | Codex adapter: invoke with `--json`, `--full-auto`, `--skip-git-repo-check` | AGENTS.md | `agents/codex.py` |
| FR-024 | Gemini adapter: invoke with `--output-format stream-json`, `--yolo` | AGENTS.md | `agents/gemini.py` |
| FR-025 | Map normalized effort level (`low`/`medium`/`high`) to agent-specific mechanism | AGENTS.md | `agents/*.py` |

### Context Pressure Monitoring

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-030 | Parse NDJSON/JSONL stream line by line concurrently with agent subprocess | SESSIONS.md | `monitor.py` |
| FR-031 | Track cumulative token usage per turn from stream events | SESSIONS.md | `monitor.py` |
| FR-032 | Compute context pressure (`estimated_tokens_used / model_context_window * 100`) after each turn | SESSIONS.md, TOKENS.md | `monitor.py` |
| FR-033 | Classify pressure into zones (green/yellow/red) using `ZoneConfig` thresholds | TOKENS.md | `monitor.py` |
| FR-034 | Support per-model zone threshold overrides from config | SESSIONS.md | `monitor.py` |
| FR-035 | Green zone: continue execution | SESSIONS.md | `monitor.py` |
| FR-036 | Yellow zone: set `kill_pending` flag; fire graceful kill at the next turn boundary. Turn boundary detection is per-agent: Claude — inter-call gap (after last `message_delta`, before next `message_start`); Codex — `turn.completed` event. Agents in degraded mode (no mid-run tokens) cannot trigger yellow mid-run. | SESSIONS.md | `monitor.py` |
| FR-037 | Red zone: trigger immediate kill | SESSIONS.md | `monitor.py` |
| FR-038 | Degraded mode: activate when (a) the agent/mode is known to lack mid-run tokens (Gemini, Claude with extended thinking — static, set at invocation) or (b) stream parsing fails mid-run for an agent that normally supports real-time monitoring (dynamic fallback — log warning, continue reading stream without pressure computation). In either case, compute pressure post-completion only; yellow zone cannot trigger mid-run. | SESSIONS.md | `monitor.py` |
| FR-039 | Run wall-clock timer concurrently; timeout triggers immediate kill with `termination_reason=timed_out` | SESSIONS.md | `monitor.py` |
| FR-040 | Timeout and pressure: whichever fires first wins, the other is cancelled | SESSIONS.md | `monitor.py` |

### Subprocess Termination

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-045 | Graceful mode: wait for current turn to complete, then signal | ARCHITECTURE.md | `runner.py` |
| FR-046 | Immediate mode: signal immediately without waiting | ARCHITECTURE.md | `runner.py` |
| FR-047 | Signal sequence: SIGINT for Codex, SIGTERM for Claude/Gemini; 5-second grace; SIGKILL if still alive. R1 ships Claude only (SIGTERM). See [ADR-006](docs/adr/ADR-006-subprocess-termination-procedure.md) | ARCHITECTURE.md | `runner.py` |
| FR-048 | Drain stdout buffer on kill — capture all complete NDJSON/JSONL lines, discard incomplete final line | ARCHITECTURE.md | `runner.py` |
| FR-049 | Record process `returncode` as-is in report | ARCHITECTURE.md | `runner.py` |

### Output Capture & Token Normalization

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-050 | Parse agent stdout, extract result text | ARCHITECTURE.md | `agents/*.py` |
| FR-051 | Normalize token usage into `NormalizedTokenUsage` dataclass | TOKENS.md | `tokens.py` |
| FR-052 | Claude: sum `input_tokens` + `cache_read_input_tokens` + `cache_creation_input_tokens` for normalized `input_tokens` | TOKENS.md, AGENTS.md | `agents/claude.py` |
| FR-053 | Codex: sum per-turn `input_tokens`/`output_tokens` deltas across all `turn.completed` events | AGENTS.md | `agents/codex.py` |
| FR-054 | Gemini: sum `prompt`/`candidates` across all models in `stats.models` | AGENTS.md | `agents/gemini.py` |
| FR-055 | Compute `total_tokens = input_tokens + output_tokens` (never from agent-reported total) | TOKENS.md | `tokens.py` |
| FR-056 | Exclude thinking/reasoning tokens from total | TOKENS.md | `tokens.py` |
| FR-057 | Track cache tokens separately (not counted toward budget) | TOKENS.md | `tokens.py` |
| FR-058 | On parse failure: capture raw stdout/stderr as `raw_output`, set `parse_error`, null `normalized_tokens` | ARCHITECTURE.md | `agents/*.py` |

### Wrap-Up Protocol

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-060 | On session end (any `termination_reason`): check for uncommitted changes in sandbox. If present, `git add -A && git commit` with message `rein: uncommitted work captured at session end (termination_reason=<reason>)`. Runs unconditionally — defense-in-depth against agents that fail to commit | SESSIONS.md | `runner.py` |
| FR-061 | Write per-run log file with termination metrics (task ID, agent, zone, tokens, turns, signal, duration) | SESSIONS.md | `runner.py` |
| FR-062 | Update PROGRESS.md with task summary (what was attempted, zone, what was captured) | SESSIONS.md | `runner.py` |
| FR-063 | Set `termination_reason` field: `completed`, `timed_out`, `context_pressure`, or `error` | SESSIONS.md | `runner.py` |

### Evaluation

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-065 | Run `validation_commands` in sandbox after agent finishes | ARCHITECTURE.md | `evaluate.py` |
| FR-066 | 60-second per-command timeout; kill on hang (SIGTERM → 5s → SIGKILL) | ARCHITECTURE.md | `evaluate.py` |
| FR-067 | Binary scoring: all commands exit 0 → score 1.0, any non-zero → score 0.0. Empty `validation_commands` → `score: null`, `validation_passed: null`. Quality gate `tests` signal gets `status: "skip"` (no commands to run). | ARCHITECTURE.md | `evaluate.py` |
| FR-068 | Capture stdout/stderr from each validation command in report | REPORTS.md | `evaluate.py` |

### Reporting

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-070 | Generate structured JSON report per run | REPORTS.md | `runner.py` |
| FR-071 | Save report to `results/{task_id}_{YYYYMMDD_HHMMSS}.json` | REPORTS.md | `runner.py` |
| FR-072 | Include `ContextPressure` data in report (zone, utilization, measurement method) | REPORTS.md, TOKENS.md | `runner.py` |
| FR-073 | Include `budget_analysis` with utilization, status, remaining, cache efficiency | REPORTS.md, TOKENS.md | `runner.py` |
| FR-074 | Report schema: `task`, `results[]`, `evaluations[]`, `timestamp` | REPORTS.md | `runner.py` |

### Token Budget

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-075 | Default budget: 70,000 tokens (input + output) | TOKENS.md | `tokens.py` |
| FR-076 | Budget status thresholds: within (<80%), warning (80-100%), exceeded (>100%) | TOKENS.md | `tokens.py` |
| FR-077 | `analyze_budget` function: returns utilization_pct, status, remaining, cache_efficiency | TOKENS.md | `tokens.py` |
| FR-078 | Budget overridable per-task (`token_budget` field) and per-run (`--budget` CLI flag) | TASKS.md | `cli.py` |

### CLI

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-080 | `rein run` command with flags: `-t`, `-a`, `-m`, `--effort`, `-o`, `--budget`, `--timeout`, `--source`, `--spec-check`, `-y`/`--yes`, `--dry-run` | ARCHITECTURE.md | `cli.py` |
| FR-080a | `--source PATH` CLI flag: overrides `workspace.source` from task JSON. Allows running the same task against different repos. Resolution: CLI `--source` > task `workspace.source` > cwd (FR-011a) | TASKS.md | `cli.py` |
| FR-081 | `rein list agents` shows available/missing agents with paths and versions | ARCHITECTURE.md | `cli.py` |
| FR-082 | `rein report` generates summary from results | ARCHITECTURE.md | `cli.py` |
| FR-083 | `--dry-run` shows what would execute without running | ARCHITECTURE.md | `cli.py` |

### Configuration

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-085 | Two-layer config: global `~/.config/rein/rein.toml` + project `.rein/rein.toml`. Optional `--config` CLI flag overrides project path | MEMORY.md | `config.py` |
| FR-085a | Parse `rein.toml` using stdlib `tomllib`. Missing file uses defaults silently. Malformed TOML fails immediately with parse error and file path. Unknown keys silently ignored (forward compatibility) | QUALITY_GATE.md | `config.py` |
| FR-085b | Config schema: `[project]`, `[defaults]`, `[spec_validation]`, `[context_pressure]`, `[models]`, `[prompt_assembly]`, `[escalation]`, `[quality_gate.*]`. Full schema in QUALITY_GATE.md | QUALITY_GATE.md | `config.py` |
| FR-085c | Resolution order: CLI flags > project `.rein/rein.toml` > global `~/.config/rein/rein.toml` > built-in defaults. For agent/model/effort: task field > CLI flag > config > defaults | MEMORY.md | `config.py` |
| FR-087 | Model context window lookup from `[models]` config section (required for Codex/Gemini) | TOKENS.md | `config.py` |
| FR-088 | Zone threshold configuration via `[context_pressure]`: global defaults + per-model overrides in `[context_pressure.overrides]` | SESSIONS.md | `config.py` |

### Release 1 Research Adoptions

| ID | Requirement | Source | Module |
|----|-------------|--------|--------|
| FR-090 | Completion promise signal: check for `.rein/complete` marker, cross-reference with validation | ROADMAP.md | `evaluate.py` |
| FR-091 | LEARNINGS.md injection: at sandbox setup, copy `.rein/LEARNINGS.md` from project root or create empty seed | ROADMAP.md | `learnings.py` |
| FR-091a | LEARNINGS.md extraction: after final verdict, diff sandbox learnings against project root, validate entries (single line, `- ` prefix, ≤ 200 chars, no duplicates), merge, warn if > 80 lines ([ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)) | ROADMAP.md | `learnings.py` |
| FR-092 | Structured escalation report on verdict "fail": per-round failure narrative, `output_excerpt` extraction (30-line heuristic), progress classification (improved/stagnated/regressed), preserved state (branch/commit/diff_stat), opt-in LLM summary enrichment via `[escalation]` config ([ADR-012](docs/adr/ADR-012-structured-escalation-report.md)) | ROADMAP.md | `escalation.py`, `runner.py` |
| FR-093 | Bounded retry cap (default 4, configurable per-task or CLI) | ROADMAP.md | `runner.py` |
| FR-094 | Defense-in-depth work protocol: structured prompt injection with commit-frequently, PROGRESS.md, LEARNINGS.md, and DEFERRED.md instructions. Context pressure is deliberately omitted from the prompt to avoid anxiety-driven shortcuts ([ADR-014](docs/adr/ADR-014-defense-in-depth-strategy.md)) | ROADMAP.md, PROMPTS.md | `prompts.py` |
| FR-094a | Seed PROGRESS.md with section template (Completed / In Progress / Blocked) and 100-line advisory cap at sandbox setup. On carry-forward retry (FR-093b), keep as-is. On fresh retry, reset to template ([ADR-015](docs/adr/ADR-015-progress-deferred-sandbox-seeding.md)) | PROMPTS.md | `sandbox.py` |
| FR-094b | Seed DEFERRED.md with header template at sandbox setup. On carry-forward retry (FR-093b), keep as-is. On fresh retry, reset to template. No size cap ([ADR-015](docs/adr/ADR-015-progress-deferred-sandbox-seeding.md)) | PROMPTS.md | `sandbox.py` |
| FR-095 | Deviation rules: 3-tier constraint structure injected into task prompt — (1) automatic fixes (bugs in own code, missing imports, deps), (2) scope boundaries (no out-of-scope files, no pre-existing issue fixes — log to `DEFERRED.md` instead, no test deletion unless explicit), (3) guard rails (3-strike rule → `DEFERRED.md`, default to not changing when unsure). Per-task custom constraints via `TaskDefinition.constraints: list[str]`, appended after defaults. Toggleable via `prompt_assembly.include_deviation_rules` in `rein.toml`. Adapted from GSD's 4-rule deviation framework ([PROMPTS.md](PROMPTS.md#4-deviation-rules-fr-095), [GSD Deep Dive](research/gsd_deep_dive/04_spec_driven_development.md#4-deviation-rules)) | ROADMAP.md, PROMPTS.md | `prompts.py` |
| FR-097 | Context reload cost tracking: `reload_tokens` field in `NormalizedTokenUsage` captures first-turn `input_tokens` as cold-start overhead. Tracked per session, accumulated across rounds/sessions, surfaced in `budget_analysis.reload_cost` block. Visibility only — does not affect budget math. See [TOKENS.md — Context Reload Cost](TOKENS.md#context-reload-cost), [ADR-010](docs/adr/ADR-010-normalized-token-usage.md) | ROADMAP.md | `tokens.py`, `agents/*.py`, `runner.py` |
| FR-098 | Pre-dispatch specification validation: prompt >= 50 chars, validation_commands non-empty. Three modes: `warn` (default, prompt operator), `strict` (block dispatch), `skip` (disable). Configurable via `--spec-check` CLI flag, `--yes` for non-interactive, and `[spec_validation]` in `rein.toml`. ([ADR-013](docs/adr/ADR-013-pre-dispatch-specification-validation.md)) | ROADMAP.md, TASKS.md | `runner.py` |

---

## 3. Non-Functional Requirements

### 3.1 Performance

- Rein overhead < 2 seconds for tempdir, < 10 seconds for worktree/copy (excluding agent execution and validation)
- Stream parsing must keep up with agent output (no backpressure)

### 3.2 Reliability

Rein's reliability model follows a defense-in-depth strategy: five independent layers (spec validation, structured prompt injection, context pressure monitoring, quality gate, graceful recovery) each catch different failure modes. No single layer is trusted — probabilistic controls (prompt instructions) reduce failure frequency, deterministic controls (monitoring, validation, recovery) guarantee detection. See [ADR-014](docs/adr/ADR-014-defense-in-depth-strategy.md).

- On agent crash: capture partial output from stream buffer, report failure with `termination_reason=error`
- On parse failure: capture raw stdout/stderr, set `parse_error`, sandbox artifacts still captured
- On disk full: standard OS errors propagate under fail-fast policy
- Sandbox cleanup guaranteed even on Rein crash (atexit / signal handler)

### 3.3 Compatibility

- Python 3.12+
- Linux, macOS, and Windows
- Git required for worktree/copy workspaces and diff capture

### 3.4 Installability

- `uv` as project manager, `pyproject.toml` with `[project.scripts]` entry mapping `rein` to CLI
- Dev install: `uv pip install -e .`
- Skip PyPI for initial release

### 3.5 Testability

- Unit tests: task loading + validation, token normalization (per-agent field mapping), zone classification logic, budget analysis
- Fixture replay tests: recorded agent NDJSON/JSONL output → parse, normalize, compute pressure, generate report (no live agent, runs in CI)
- CLI integration tests: full end-to-end with live Claude Code (manual, `@pytest.mark.integration`)

---

## 4. Error Handling

| Failure Mode | Behavior |
|---|---|
| Agent CLI not in PATH | `is_available()` pre-flight check fails immediately, suggests `rein list agents` |
| Agent not authenticated | Surfaces as subprocess error, stderr captured in report |
| Malformed task JSON | Schema validation at load time, fail before sandbox creation |
| NDJSON/JSONL parse failure | Raw stdout/stderr as `raw_output`, `parse_error` set, `normalized_tokens` null, score 0.0 |
| Agent timeout | Subprocess termination procedure (immediate): SIGTERM/SIGINT → 5s → SIGKILL |
| Context pressure kill | Yellow: graceful (wait for turn). Red: immediate. Both: wrap-up protocol runs. |
| Validation command hangs | 60-second per-command timeout, same kill sequence |
| Disk full | OS errors propagate under fail-fast policy |

---

## 5. Acceptance Criteria

**Demo scenario:** `rein run -t tasks/example_fizzbuzz.json -a claude` completes the full lifecycle — task loaded, sandbox created, agent invoked, stream parsed in real-time with context pressure computed per turn, output parsed, tokens normalized, validation commands pass, JSON report written to `results/` including `ContextPressure` data.

**Tests that must pass:**
- Unit tests: task loading + validation, token normalization, zone classification, budget analysis
- Fixture replay tests: recorded agent output → full pipeline (no live agent, runs in CI)
- CLI integration test: full end-to-end with live Claude Code (manual, `@pytest.mark.integration`)

**Example tasks (3, all must pass integration):**
1. Greenfield simple — fizzbuzz (tempdir, trivial validation)
2. Greenfield with dependencies — Flask REST API (tempdir, venv setup, pytest validation)
3. Brownfield — worktree task against a bundled test repo (worktree workspace, diff capture, existing code context)

---

## 6. Assumptions & Constraints

- Agents are already installed and authenticated by the user
- Git is required for worktree/copy workspaces and diff capture
- Python 3.12+
- Linux, macOS, and Windows are all targeted
- Internet connectivity depends on the agent, not Rein (Rein makes no network calls)
- No root/admin privileges required

---

## 7. Success Metrics

- **Rein overhead:** < 2 seconds for tempdir, < 10 seconds for worktree/copy (excluding agent execution and validation)
- **Token normalization accuracy:** 0% deviation — exact match with raw agent-reported values
- **Task types supported:** 3 end-to-end (greenfield simple, greenfield with deps, brownfield worktree)
- **Report schema compliance:** 100%, zero validation errors across all test runs

---

## 8. Resolved Decisions

| Issue | Decision |
|---|---|
| Greenfield-only vs. brownfield | Brownfield ships in Release 1. All 3 workspace types. |
| Cache token semantics | Cached tokens included in `input_tokens`. Claude adapter sums all three partition fields. |
| Codex bare sandbox failure | Always pass `--skip-git-repo-check`. |
| Positioning vs. comparison | Comparison is a capability, not the purpose. |
| Context window monitoring | Ships in Release 1. Defining feature. |
| Codex/Gemini turn limits | Leave to agent defaults. Context pressure is the control mechanism. |

---

## 9. Release Scope

See [ROADMAP.md](ROADMAP.md) for phased releases (R1/R2/R3/Future) and implementation modules ordered by dependency.
