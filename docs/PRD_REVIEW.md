# PRD Review — 2026-03-08

Review of `PRD_SKETCH.md` prior to implementation.

---

## Strengths

1. **Traceability** — every FR links back to source doc and target module.
2. **Error handling matrix (S4)** covers the right failure modes with clear behaviors.
3. **Acceptance criteria (S5)** are concrete and testable — three task types covering greenfield/brownfield.
4. **Resolved decisions (S8)** prevents re-litigation of settled questions.
5. **Defense-in-depth framing (S3.2)** with the probabilistic vs. deterministic control distinction is sharp.

---

## Gaps — Missing FRs

### G-1: Retry orchestration — RESOLVED

FR-093 defines the retry cap (default 4) but not the retry loop itself. Resolved through discussion — proposed FRs below.

#### Proposed FRs

| ID | Requirement | Module |
|----|-------------|--------|
| FR-093a | Retry triggers: retry on any non-`completed` termination (`error`, `context_pressure`, `timed_out`) and on `completed` with verdict "fail". The bounded retry cap (FR-093) is the sole guard against waste. | `runner.py` |
| FR-093b | Sandbox state policy: if validation ran and failed (verdict "fail", validation commands returned non-zero), carry forward the sandbox into the next round. On all other outcomes (validation couldn't run, agent crash, context pressure kill, timeout), create a fresh sandbox. | `runner.py`, `sandbox.py` |
| FR-093c | Cross-round knowledge injection: before each retry, inject into the prompt: (1) updated LEARNINGS.md extracted from the failed round (per FR-091a), (2) condensed failure narrative — progress classification, plain-language termination reason, validation output truncated to 30-line excerpt, `git diff --stat` summary of files touched. | `runner.py`, `prompts.py` |
| FR-093d | Operator vs. agent reporting split: the full escalation report (FR-092) is written to the JSON report for the operator. The agent receives only the condensed failure narrative from FR-093c. | `escalation.py`, `prompts.py` |

### G-2: PROGRESS.md / DEFERRED.md sandbox seeding — RESOLVED

Resolved via [ADR-015](adr/ADR-015-progress-deferred-sandbox-seeding.md). Both files seeded at sandbox setup with minimal templates. PROGRESS.md capped at 100 lines (advisory). DEFERRED.md uncapped. Both start empty every run (no project root seeding). Carry-forward follows FR-093b sandbox state policy. No write-back to project root in R1.

#### Proposed FRs

| ID | Requirement | Module |
|----|-------------|--------|
| FR-094a | Seed PROGRESS.md with section template (Completed / In Progress / Blocked) and 100-line advisory cap at sandbox setup. On carry-forward retry (FR-093b), keep as-is. On fresh retry, reset to template. | `sandbox.py` |
| FR-094b | Seed DEFERRED.md with header template at sandbox setup. On carry-forward retry (FR-093b), keep as-is. On fresh retry, reset to template. No size cap. | `sandbox.py` |

### G-3: Config file parsing and schema — RESOLVED

FR-085/086 describe the two-layer config and resolution order, but no FR covers parsing or the full schema. Format mismatch (I-1) resolved: `rein.toml` everywhere.

#### Proposed FRs

| ID | Requirement | Module |
|----|-------------|--------|
| FR-085a | Parse `rein.toml` using stdlib `tomllib`. Two locations: global `~/.config/rein/rein.toml`, project `.rein/rein.toml`. Optional `--config` CLI flag overrides project path. Missing file is not an error — use defaults. Malformed TOML fails immediately with parse error and file path. Unknown keys are silently ignored (forward compatibility). | `config.py` |
| FR-085b | Config schema with sections: `[project]` (name), `[defaults]` (agent, model, effort, token_budget, timeout_seconds, max_rounds), `[spec_validation]` (mode, min_prompt_length, require_validation_commands), `[context_pressure]` (green_max_pct, yellow_max_pct, per-model overrides), `[models]` (per-model context_window), `[prompt_assembly]` (include_preamble, include_work_protocol, include_deviation_rules, include_completion_signal), `[escalation]` (summary_agent, summary_model, summary_token_budget, summary_timeout_seconds), `[quality_gate.*]` (per-signal: enabled, required, commands; plus signal-specific keys). Full schema documented in QUALITY_GATE.md. | `config.py` |
| FR-085c | Resolution order: CLI flags > project `.rein/rein.toml` > global `~/.config/rein/rein.toml` > built-in defaults. For agent/model/effort: task field > CLI flag > config > defaults. | `config.py` |

### G-4: Workspace source path — RESOLVED (downgraded to enhancement)

TASKS.md already defines `workspace.source` as a task-level field with JSON schema validation (required for worktree/copy). The core requirement is covered. Two convenience enhancements added:

#### Proposed FRs

| ID | Requirement | Module |
|----|-------------|--------|
| FR-080a | `--source PATH` CLI flag: overrides `workspace.source` from task JSON. Allows running the same task definition against different repos without editing the JSON. Resolution: CLI `--source` > task `workspace.source`. | `cli.py` |
| FR-011a | Default source to cwd: if `workspace.type` is `worktree` or `copy` and neither CLI `--source` nor task `workspace.source` is provided, default to the current working directory. Fail if cwd is not a git repo for worktree type. | `sandbox.py` |

### G-5: Diff capture mechanism — RESOLVED

FR-015 lacked specifics on baseline, command, and output destination. Resolved with two new FRs.

Artifacts defined as: (1) full unified diff patch, (2) diff stat summary, (3) final commit SHA. Baseline is a pinned commit SHA captured at sandbox creation — not a branch name or HEAD reference (source repo may receive commits during the run). For tempdir, an initial commit is created after file seeding and setup_commands to establish the baseline.

Diff patch and stat go into the JSON report. Patch also written as a standalone `.patch` file in the results directory (survives sandbox cleanup for tempdir/copy). Condensed failure narrative (FR-093c) uses only the stat.

#### Proposed FRs

| ID | Requirement | Module |
|----|-------------|--------|
| FR-015a | Record baseline commit SHA at sandbox creation time (before agent invocation). For tempdir: `git init && git add -A && git commit` after file seeding and setup_commands. For worktree/copy: capture `HEAD` SHA at creation time. Store as `baseline_sha` in run context. | `sandbox.py` |
| FR-015b | After agent finishes (before cleanup): run `git diff <baseline_sha> HEAD` for full patch and `git diff --stat <baseline_sha> HEAD` for summary. Store both in report as `diff_patch` and `diff_stat` fields. Write patch to `results/{task_id}_{timestamp}.patch`. | `sandbox.py`, `runner.py` |

---

## Inconsistencies

### I-1: Config format mismatch — RESOLVED

Resolved in G-3. Format is `rein.toml` everywhere. FR-085/086 in PRD_SKETCH.md updated.

### I-2: FR-060 scope ambiguity — RESOLVED

FR-060 now runs unconditionally on any termination reason (defense-in-depth). Checks `git status` first — no-op if nothing uncommitted. Commit message uses `rein:` prefix with termination reason to distinguish Rein's safety commit from the agent's deliberate work.

### I-3: FR-047 signal mapping includes non-MVP agents — RESOLVED

Added R1 scope note to FR-047 in PRD_SKETCH.md. The general rule (per-agent signal) is correct and stays — just annotated that R1 ships Claude only (SIGTERM).

### I-4: FR-091 / FR-091a sub-ID numbering — CLOSED (no longer applicable)

Sub-IDs are now the established convention across the PRD (FR-011a, FR-015a/b, FR-080a, FR-085a/b/c, FR-093a/b/c/d, FR-094a/b). FR-091a is consistent, not anomalous.

---

## Specification Tightness

### S-1: FR-036 "graceful kill" mechanism undefined — RESOLVED

Turn boundary detection is per-agent. Monitor sets `kill_pending` flag on yellow; fires kill at the next turn boundary. For Claude Code: inter-call gap (after last `message_delta`, before next `message_start`). For Codex: `turn.completed` event. Degraded-mode agents (no mid-run tokens) cannot trigger yellow mid-run — pressure computed post-completion only. FR-036 and SESSIONS.md Zone Actions updated.

### S-2: FR-038 "degraded mode" activation criteria — RESOLVED

Two trigger conditions: (1) **static** — agent/mode known to lack mid-run tokens (Gemini, Claude with extended thinking), set at invocation; (2) **dynamic** — stream parse failure mid-run for an agent that normally supports real-time monitoring, logged as warning, falls back to post-completion pressure computation. FR-038, SESSIONS.md, and ARCHITECTURE.md updated.

### S-3: Empty validation_commands scoring — RESOLVED

Empty `validation_commands` → `score: null`, `validation_passed: null` (no validation ran — distinct from pass or fail). Quality gate `tests` signal gets `status: "skip"`. Completion confidence matrix expanded from 4 to 6 outcomes: added `"unverified"` (promise + no validation) and `"unevaluated"` (no promise + no validation). Escalation report accepts `"unverified"` as a third confidence value. Updated in FR-067, ADR-002, ADR-012, ARCHITECTURE.md, REPORTS.md, QUALITY_GATE.md.

### S-4: FR-092 "30-line heuristic" unspecified — RESOLVED

Already specified in [ADR-012](adr/ADR-012-structured-escalation-report.md) §`output_excerpt` extraction: scan for failure markers (`FAILED`, `Error:`, `AssertionError`, etc.) → 30 lines from first marker; no marker → last 30 lines; empty output → `null`.

### S-5: FR-021 git identity source — RESOLVED

Already specified in [ADR-001](adr/ADR-001-agent-git-identity.md): name is deterministically derived from agent/model/effort (`claude-code/opus/high`), email is fixed `agent@rein.local`. No config keys needed — identity is a function of parameters that are already configurable. Rein safety commits (FR-060) reuse the agent identity with `rein:` commit message prefix.

---

## Platform Concern — RESOLVED

R1 scoped to Linux and macOS. Windows deferred to R2. The POSIX signal termination model (SIGTERM → 5s grace → SIGKILL) has no direct Windows equivalent — `TerminateProcess` is uncatchable (equivalent to SIGKILL), and graceful stop requires `CTRL_BREAK_EVENT` with a different strategy. Updated PRD_SKETCH.md S3.3 and S6, ARCHITECTURE.md.

---

## Minor Nits

- ~~S1 repeats "comparison is a byproduct" from BRIEF.md~~ Resolved: merged into one sentence.
- S7 "0% deviation" for token normalization is a tautology when summing the agent's own fields. The metric should measure *completeness* (no dropped events), not accuracy.

---

## Recommendations (Priority Order)

1. Add FR cluster for retry orchestration (G-1)
2. Reconcile config format to `rein.toml` everywhere (I-1)
3. Add FR for workspace source path (G-4)
4. ~~Tighten FR-036/038 with one sentence each on mechanism (S-1, S-2)~~ Resolved.
5. Add FR for PROGRESS.md / DEFERRED.md seeding (G-2)
6. ~~Specify empty-validation scoring behavior (S-3)~~ Resolved.
7. ~~Address Windows process termination or scope R1 to POSIX~~ Resolved: R1 scoped to Linux/macOS.
