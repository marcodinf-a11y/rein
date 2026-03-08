# MVP Value Synthesis

Cross-cutting analysis of all research deep dives and proposals (~100 documents, 26 proposals) to identify what is valuable for rein MVP.

Sources: Stripe Minions (8 docs), Goose (11 docs), RLM (11 docs), Ralph Wiggum (7 docs), Ralph Orchestrator (12 docs + 6 proposals), Principal Skinner (7 docs + 5 proposals), Gas Town (6 docs + 6 proposals), GSD (6 docs + 5 proposals), Landscape Research (12 docs).

---

## Design Validations

These confirm Rein's existing architecture is correct. No changes needed — just confidence.

| Design Choice | Validated By |
|---|---|
| External orchestrator (zero agent context overhead) | GSD (10-15% in-process overhead, unmeasured), Gas Town (15-30% LLM supervisor waste) |
| Zone-based kill over advisory warnings | GSD (warns but can't kill), Goose (80% compaction is blunt) |
| Subprocess isolation over stop hooks | Ralph Wiggum (Huntley warns against stop hooks), Ralph Orchestrator (PTY complicates parsing) |
| Operator-defined validation commands | Ralph Wiggum (agent self-eval enables reward hacking), Stripe (deterministic gates > LLM judgment) |
| Binary eval via validation commands | Gas Town (LLM-as-supervisor = 15-30% waste), Context Degradation research |
| Piped stdout over PTY | Ralph Orchestrator (escape codes break parsing) |
| Sandbox isolation (worktree/tempdir) | Goose (no Linux sandbox at all), Stripe (filesystem as state store) |
| Snapshot-per-execution reporting | GSD (STATE.md drifted from reality after 18 phases — agent-maintained state accumulates errors) |

---

## Tier 1: Adopt for MVP (high-impact, low-effort)

### 1. Agent Git Identity

**Source:** Principal Skinner proposal 02

Set `GIT_AUTHOR_NAME="{agent}/{model}/{effort}"` and `GIT_AUTHOR_EMAIL="agent@rein.local"` in the subprocess environment before launch. Enables `git log --author` filtering and `git blame` attribution in the sandbox.

**Cost:** Two env vars. Zero infrastructure.

### 2. Completion Promise Signal

**Source:** Ralph Orchestrator proposal 04

Agent writes a `.rein/complete` marker file when it believes it is done. Cross-referencing with validation commands yields four outcomes:

| Promise | Validation | Verdict |
|---|---|---|
| Yes | Pass | Confident |
| No | Pass | Suspicious |
| Yes | Fail | Overconfident |
| No | Fail | Incomplete |

The "overconfident" case directly detects hallucinated success and reward hacking.

**Cost:** File existence check + prompt suffix instruction.

### 3. LEARNINGS.md Seed File with Size Cap

**Source:** Ralph Orchestrator proposal 05, GSD proposal 02

Per-workspace file of operational knowledge (build commands, compiler quirks, test patterns) that persists across sessions. Eliminates 1-3 wasted turns per session where the agent rediscovers environment facts.

Hard cap: 80 lines (GSD: "a DIGEST, not an archive"). Prompt instruction to evict oldest entries before adding new ones. Without the cap, agents append indefinitely until the file itself causes context pressure.

Operator review between sessions is recommended — agent-written content is not inherently trustworthy.

**Cost:** A few hundred tokens per session.

### 4. Structured Escalation Report on Failure

**Source:** Stripe deep dive (07)

On `verdict: "fail"`, produce a structured escalation report: what was attempted, what failed, diagnostic summary, agent's branch and changes preserved. Do not just record the failure — produce actionable context for the human.

**Cost:** Template formatting of existing data.

### 5. Pre-Dispatch Specification Validation

**Source:** Gas Town proposal 03
**Status:** Adopted → [ADR-013](../docs/adr/ADR-013-pre-dispatch-specification-validation.md), [TASKS.md](../TASKS.md#pre-dispatch-specification-validation)

Two deterministic checks before spending tokens:

1. Prompt >= 50 chars
2. `validation_commands` non-empty

Three modes: `warn` (default, prompt operator), `strict` (block dispatch), `skip` (disable). Configurable via `--spec-check` CLI flag and `[spec_validation]` in `rein.toml`. `--yes` flag for non-interactive/CI use.

Acceptance criteria heuristic was dropped — regex keyword matching is brittle and gameable. Gas Town has the `AcceptanceCriteria` field but doesn't enforce it; neither should we. The two concrete checks catch the worst offenders.

Gas Town's biggest operational lesson: "Planning and design become the limiting factor" (Appleton). Confirmed by Gas Town source code review: zero pre-dispatch spec quality checks exist.

**Cost:** Two string/list checks.

### 6. Bounded Retry Cap (Default = 2)

**Source:** Stripe deep dive (01)

"If the LLM can't fix it in two tries, a third won't help. It just burns compute." Independent research confirms diminishing returns after 1-2 attempts on the same error.

Configurable via task definition or CLI flag. Default: 2.

**Cost:** Counter + early exit.

### 7. Defense-in-Depth Prompt Instruction

**Source:** GSD (02)

Add to the agent's task prompt: "If context pressure is high, commit current work and write a progress summary immediately." The agent may save state voluntarily before rein kills externally.

GSD's context monitor warns at 65%/75% — nearly identical to Rein's yellow/red zones. Key difference: GSD cannot kill. Rein can — this prompt instruction is a complementary soft signal, not a replacement for hard kill.

**Cost:** One paragraph in task prompt template.

### 8. Deviation Rules in Task Prompts

**Source:** GSD (04)

Include concrete deviation rules in task prompts:

- Auto-fix bugs caused by the current task
- Auto-add critical safety (auth, validation) if missing
- Auto-fix blocking dependencies
- STOP for architectural changes (new DB table, schema change, new service)
- "3 auto-fix attempts per task, then document and continue"
- "5+ consecutive Read/Grep/Glob without Edit/Write/Bash → STOP and report"

**Cost:** Prompt template additions.

### 9. Network Isolation Flag

**Source:** Principal Skinner proposal 01

Optional `--network-isolate` flag using bubblewrap `--unshare-net` or Docker `--network none`. Closes the data exfiltration gap — the strongest argument Principal Skinner makes against containment-only safety.

**Cost:** One CLI flag, one subprocess argument.

### 10. Context Reload Cost Tracking

**Source:** Gas Town (03)

Every session after a context pressure kill pays a cold-start tax (codebase re-read, task definition re-parse, role instructions re-ingested). Track this separately in token accounting so operators can see the true cost of context rotation.

**Cost:** Separate counter in `NormalizedTokenUsage`.

---

## Tier 2: Adopt for MVP (medium-effort, high-value)

### 11. Observation Masking

**Source:** RLM deep dive (01, 08), Context Degradation research (02)

Truncate prior tool outputs from the agent's conversation while preserving reasoning traces. NeurIPS 2025 DL4Code paper (Lindenbauer et al.): "approximately 50% cost reduction without quality loss" for SE agents. Simple truncation outperforms LLM summarization.

Applies to all agents, no RLM infrastructure needed. Implementation depends on agent-specific mechanisms (Claude's context editing reduced consumption by 84% in 100-turn eval).

**Cost:** Agent-specific integration work. May require hooks or adapter-level truncation.

### 12. Crash-Recoverable Task State

**Source:** Gas Town proposal 02

Write `.rein/task_state.json` at lifecycle checkpoints (dispatch, zone change, session end, validation pass). Git-commit alongside agent work for atomic durability. On restart, check for `status=in_progress` and resume from `last_good_commit`.

Zone-based kills are "controlled crashes" — they need the same recovery path as actual crashes.

**Cost:** JSON write at ~5 lifecycle points, resume logic in runner.

### 13. Multi-Run Evaluation

**Source:** RLM deep dive (02, 08)

RLM reproduction study: 0/6 to 6/6 variance across 30 identical runs. Single-run binary evaluation is unreliable for any non-deterministic agent.

Add `rein run --runs N` (default 1). Run each task N times, report pass rate + variance + cost distribution. Even `--runs 3` dramatically improves confidence.

**Cost:** Loop wrapper + statistical summary in report.

### 14. Deterministic/Agentic Interleaving in Quality Gate

**Source:** Stripe deep dive (01)

Stripe's Blueprint wraps LLM calls in deterministic steps: `run_lint() -> if autofix_available: apply_autofix() else: agent_fix() -> run_ci() -> repeat (max 2)`.

Rein should run lint/test/build as deterministic commands in the quality gate, only consuming an agent round when the fix requires judgment. An `autofix` mode for commands like `lint --fix` or `cargo fmt` applies fixes without burning a round.

**Cost:** Conditional logic in quality gate pipeline.

### 15. Goose Adapter via goosed REST API

**Source:** Goose deep dive (01, 10)

Use `goosed` REST API (`POST /reply` with SSE streaming, `X-Secret-Key` auth) over CLI stdout parsing. More structured, less fragile. SSE events are typed: `Message`, `Error`, `Finish`, `ModelChange`, `Notification`, `Ping`. Parse `ProviderUsage` from response metadata for cost tracking.

Goose CLI exit codes are unreliable — parse stream events instead.

**Cost:** HTTP client + SSE parser in Goose adapter.

---

## Future: Post-MVP

Organized by the milestone they become relevant.

### Multi-Session Mode

| Pattern | Source | Description |
|---|---|---|
| Stagnation detection | Ralph Orchestrator proposal 02 | Three detectors: stale (same failure 3x), oscillation (pass/fail alternation 3x), crash loop (3 consecutive non-zone exits). Stop retries, escalate |
| Inter-session validation | Ralph Orchestrator proposal 03 | Run all prior tasks' `validation_commands` after each session. Catches regressions before wasting compute |

### Multi-Task Mode

| Pattern | Source | Description |
|---|---|---|
| Wave-based execution | GSD proposal 03 | `depends_on: list[str]` in TaskDefinition. Topological sort into waves. Parallel within waves, sequential across. Validate after each wave |
| Workflow persistence | Gas Town proposal 05 | Flat DAG persisted in `.rein/workflow_state.json`. On crash, skip completed tasks, resume in-progress |
| Reusable task templates | Gas Town proposal 06 | YAML with `${parameter}` substitution. Role-based model selection (Opus for design, Sonnet for implementation) |
| Pre-dispatch validation for auto-generated tasks | GSD proposal 04 | Completeness, actionability, scope sizing, dependency correctness, validation coverage |

### Multi-Agent Mode

| Pattern | Source | Description |
|---|---|---|
| Merge queue | Gas Town proposal 04 | Worktree per agent, sequential rebase, validation after merge, operator-resolved conflicts. 2-5 concurrent agents max |
| Lead-Worker model routing | Goose (04) | Frontier model for initial generation, cheaper model for iterative CI fixes |
| Sub-query fan-out | RLM / Prime Intellect (06) | Parallel sub-queries within a single task, configurable parallelism |

### Observability Phase

| Pattern | Source | Description |
|---|---|---|
| JSONL event stream | Ralph Orchestrator proposal 06 | Timeline complement to JSON report. Events: zone_change, session_kill, stream_parse. ~1 event/turn |
| Action frequency monitoring | Principal Skinner proposal 04 | Count tool calls by category (bash, file_writes, network, destructive). Configurable thresholds |
| Arize Phoenix tracing | Landscape 11 | Fully open-source (Apache 2.0), accepts OTLP traces, all features free |

### Eval Hardening

| Pattern | Source | Description |
|---|---|---|
| Multi-run adversarial evaluation | Principal Skinner proposal 05 | `rein eval --runs 10 --vary-prompt`. Pass rate, failure mode distribution, cost distribution across prompt variations |
| Trace grading | Landscape 04, 07 | Score the full execution log (decisions, tool calls, reasoning), not just final output |

---

## Universal Avoids

Confirmed across multiple independent sources. These are not "maybe later" — they are structurally wrong for this project.

| Never Do | Why | Sources |
|---|---|---|
| In-process orchestration | Degrades under same pressure it manages | GSD, RLM, Gas Town |
| LLM-as-supervisor | 15-30% token waste, no deterministic quality guarantee | Gas Town, Stripe |
| Advisory-only context warnings | Degraded agent ignores them | GSD, Goose |
| Agent self-evaluation as sole judge | Reward hacking: delete tests, stub implementations, add skip annotations | Ralph Wiggum, Stripe |
| Fork an agent runtime | Maintenance bomb against fast-moving upstream (Goose: 2+ releases/week) | Stripe |
| Trust advertised context windows | MECW can be 99% smaller than MCW; Gemini degrades earliest despite 2M window | Context Degradation, Landscape 05 |
| Infinite/unbounded loops | No convergence guarantee; "deterministically bad" per creator | Ralph Wiggum |
| DAG schedulers / complex orchestration | Sequential pipeline with bounded loops is sufficient for all studied systems | Stripe, Gas Town |
| RLM integration now | Zero action-oriented coding evidence; all benchmarks are read-only comprehension | RLM |
| 20+ concurrent agents | Unproven quality at scale, super-linear cost, $2-5K/month waste reported | Gas Town |
| Skip-permissions as default | Amplifies blast radius with multiple agents | GSD |
| Depth > 1 recursion | ~20% degradation per level, compounds; depth-2 dropped 42.1% to 33.7% | RLM |
| Markdown-only reports | Not machine-parseable; structured JSON is non-negotiable for cross-run comparison | Ralph Orchestrator |
| PTY-based agent execution | Terminal escape codes break stream parsing | Ralph Orchestrator |
| LLM-based merge resolution | "Creatively reimagines implementations" with no quality gate | Gas Town |
| Docker alone for security | Three critical runC vulnerabilities Nov 2025 | Landscape 06 |

---

## Source Cross-Reference

| Deep Dive | Documents | Proposals | Key Contribution |
|---|---|---|---|
| Stripe Minions | 8 (00-07) | — | Deterministic/agentic interleaving, bounded retries, structured escalation |
| Goose | 11 (00-10) | — | Adapter strategy (goosed API), sandbox gaps, context management anti-patterns |
| RLM | 11 (00-08 + 2) | — | Observation masking, multi-run evaluation, recursion avoidance |
| Ralph Wiggum | 7 (00-06) | — | Context rotation validation, completion promise concept, failure modes |
| Ralph Orchestrator | 6 (00-05) | 6 | Completion promise signal, LEARNINGS.md, stagnation detection, event stream |
| Principal Skinner | 7 (00-06) | 5 | Agent git identity, network isolation, action frequency, OWASP mapping |
| Gas Town | 6 (00-05) | 6 | Crash-recoverable state, spec validation, merge queue, reload cost |
| GSD | 6 (00-05) | 5 | Orchestrator budget ceiling, size-capped state, deviation rules, wave execution |
| Landscape | 12 (01-12) | — | Context degradation evidence, sandbox tiers, eval benchmarks, tracing tools |
