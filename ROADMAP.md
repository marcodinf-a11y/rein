# Rein — Roadmap

Release planning and phased scope for Rein. For what the system *is* and *does*, see [BRIEF.md](BRIEF.md) and [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Release 1 — Single-Agent Workflow

The initial release. One task, one agent, full lifecycle.

**Core:**
- Claude Code adapter (`claude` CLI, `--output-format stream-json`)
- All 3 workspace types: tempdir, worktree, copy
- Real-time context pressure monitoring (stream parse, zone classification, kill signaling)
- JSON task format with schema validation
- Quality gate with configurable signals (validation commands, binary scoring)
- Structured JSON reports

**Included from research (low-effort, high-impact):**
- ~~Agent git identity (`GIT_AUTHOR_NAME="{agent}/{model}/{effort}"`)~~ → documented in [ARCHITECTURE.md](ARCHITECTURE.md#agent-git-identity)
- ~~Completion promise signal (`.rein/complete` marker)~~ → documented in [REPORTS.md](REPORTS.md#completion-confidence), [ADR-002](docs/adr/ADR-002-completion-promise-signal.md)
- ~~LEARNINGS.md seed file with 80-line cap~~ → documented in [ARCHITECTURE.md](ARCHITECTURE.md#learningsmd-injection-and-extraction), [PROMPTS.md](PROMPTS.md#3-work-protocol-defense-in-depth--fr-094), [ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)
- ~~Structured escalation report on failure~~ → documented in [QUALITY_GATE.md](QUALITY_GATE.md#escalation-report), [REPORTS.md](REPORTS.md#escalation-report), [ADR-012](docs/adr/ADR-012-structured-escalation-report.md)
- ~~Pre-dispatch specification validation~~ → documented in [TASKS.md](TASKS.md#pre-dispatch-specification-validation), [ADR-013](docs/adr/ADR-013-pre-dispatch-specification-validation.md)
- Bounded retry cap (default 2)
- Defense-in-depth prompt instruction (context pressure awareness)
- Deviation rules in task prompts
- Network isolation flag (`--network-isolate`)
- Context reload cost tracking

**Out of scope:** Codex/Gemini adapters, post-kill summary agent, parallel dispatch, YAML tasks, session resume, multi-run evaluation.

---

## Release 2 — Multi-Agent & Evaluation

Expand agent support, add YAML, improve evaluation confidence.

- Codex CLI adapter (`codex exec --json`, JSONL parsing)
- Gemini CLI adapter (`gemini --output-format stream-json`)
- YAML task format (multiline block scalars, ~24% fewer tokens)
- Multi-run evaluation (`rein run --runs N`, pass rate + variance + cost distribution)
- Session resume (restart from last checkpoint after context pressure kill)
- Stagnation detection (stale failure 3x, oscillation 3x, crash loop 3 consecutive)
- Crash-recoverable task state (`.rein/task_state.json` at lifecycle checkpoints)
- Observation masking (truncate prior tool outputs, preserve reasoning traces)
- Deterministic/agentic interleaving in quality gate (lint `--fix` before agent round)

---

## Release 3 — Composition & Parallelism

Multi-agent workflow composition and concurrent execution.

- Multi-agent composition (different agents/models for different roles)
- Parallel dispatch (`--parallel` flag, concurrent task execution)
- Wave-based execution (`depends_on` field, topological sort, parallel within waves)
- Workflow persistence (`.rein/workflow_state.json`, skip completed on restart)
- Merge queue (worktree per agent, sequential rebase, validation after merge)
- Reusable task templates (YAML with `${parameter}` substitution)

---

## Future

Longer-term capabilities, not yet scheduled.

**Adapters:**
- Goose adapter via `goosed` REST API (SSE streaming, `ProviderUsage` cost tracking)

**Observability:**
- JSONL event stream (timeline complement to JSON report: zone_change, session_kill, stream_parse)
- Action frequency monitoring (tool call counts by category, configurable thresholds)
- Arize Phoenix tracing (Apache 2.0, OTLP traces)

**Eval hardening:**
- Multi-run adversarial evaluation (`rein eval --runs 10 --vary-prompt`)
- Trace grading (score full execution log, not just final output)

**Multi-agent scaling:**
- Lead-Worker model routing (frontier model for generation, cheaper for CI fixes)
- Sub-query fan-out (parallel sub-queries within a single task)

---

## Implementation Modules

Ordered by dependency. From `response.md` analysis — zero source code exists yet.

| Module | What it does | Complexity |
|--------|-------------|------------|
| `models.py` | `TaskDefinition`, `WorkspaceConfig`, `NormalizedTokenUsage`, `ContextPressure`, `ZoneConfig`, `BudgetStatus` dataclasses | Low |
| `tokens.py` | `analyze_budget`, zone logic | Low |
| `agents/base.py` | `AgentAdapter` protocol | Low |
| `cli.py` | Click CLI (`rein run`, `rein list`), `--spec-check`/`--yes` flags | Low |
| `spec_validation.py` | Pre-dispatch spec checks (prompt length, validation commands), mode handling ([ADR-013](docs/adr/ADR-013-pre-dispatch-specification-validation.md)) | Low |
| `evaluate.py` | Run validation commands, binary scoring | Low |
| `sandbox.py` | tempdir/worktree/copy creation, file seeding, setup commands, cleanup | Medium |
| `learnings.py` | LEARNINGS.md injection (copy to sandbox), extraction (diff, validate, merge back), structural validation | Low |
| `escalation.py` | Escalation report on terminal failure: mechanical summary, output excerpt extraction, progress classification, opt-in LLM enrichment ([ADR-012](docs/adr/ADR-012-structured-escalation-report.md)) | Low |
| `agents/claude.py` | Invoke `claude -p --output-format stream-json`, parse NDJSON, normalize tokens | Medium |
| `monitor.py` | Real-time stream parsing, zone classification, kill signaling | High |
| `runner.py` | Orchestrator: sandbox, invoke, monitor, capture, evaluate, report | High |

---

## Open Decisions

Captured from gap analysis. Must be resolved before or during implementation.

| Decision | Options | Context |
|----------|---------|---------|
| Cache token semantics | Include in `input_tokens` or separate field | Per-agent behavior differs |
| Codex bare sandbox | Always `--skip-git-repo-check` or require `git init` in setup | Codex fails without git repo |
| Codex/Gemini turn limits | Enforce max turns or leave to agent defaults | Affects context pressure accuracy |

See [PRD_SKETCH.md](PRD_SKETCH.md) for the full requirements document.
