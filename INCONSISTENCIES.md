# Rein — Inconsistency & Gap Analysis

Cross-document review of all seven specification files, with focus on greenfield vs brownfield support, internal contradictions, and undocumented edge cases.

---

## Critical

### ~~1. `total_tokens` is always 0 as documented~~ — RESOLVED

**Status:** Resolved. TOKENS.md now contains the canonical `NormalizedTokenUsage` definition with `__post_init__` that computes `total_tokens = input_tokens + output_tokens` via `object.__setattr__`. ARCHITECTURE.md no longer duplicates the definition — it defers to TOKENS.md. All three adapter functions in AGENTS.md correctly omit `total_tokens` from their constructors, relying on `__post_init__`.

---

### ~~2. Sandbox model has no brownfield story~~ — RESOLVED

**Status:** Resolved. `WorkspaceConfig` dataclass added to TASKS.md with three sandbox types: `tempdir` (default, greenfield), `worktree` (git worktree from existing repo), and `copy` (copy source tree into temp directory). The `workspace` field is optional on `TaskDefinition` — omitting it preserves the existing `tempdir` behavior. TASKS.md documents workspace types with diff baseline semantics and cleanup behavior for each. JSON schema updated with conditional `source` requirement. Brownfield examples added (JSON and YAML). ARCHITECTURE.md Execution Isolation section updated with a workspace type table. Execution flow step 2 and the system diagram reflect the three workspace types. SESSIONS.md Session Lifecycle step 2 updated.

---

### ~~3. Codex will fail in bare sandboxes~~ — RESOLVED

**Status:** Resolved. The Codex invocation now always includes `--skip-git-repo-check` since sandboxes may not contain a `.git` directory. The `_parse_codex_jsonl` parser now handles `turn.failed` and `error` event types, collecting error messages into an `errors` list on the result dict. The Codex quirks section documents why the flag is always passed.

---

## High

### ~~4. Cache token semantics are ambiguous — budget math may be wrong~~ — RESOLVED

**Status:** Resolved. Per-provider cache semantics researched and documented in TOKENS.md Cache Semantics section. Claude's `input_tokens` is **exclusive** of cache (partitioned model — three fields sum to total input). Codex and Gemini are **inclusive** (cached tokens are a subset of the input field). The Claude adapter in AGENTS.md now sums `input_tokens + cache_read_input_tokens + cache_creation_input_tokens` into the normalized `input_tokens`, making it consistent with Codex and Gemini. The normalized `input_tokens` always means "total input tokens processed" across all agents. TOKENS.md field mapping table updated to reflect the summation.

---

### ~~5. "Not a benchmarking tool" contradicts its own design~~ — RESOLVED

**Status:** Resolved. Comparison features removed from the design entirely. The "Comparison" use case in BRIEF.md replaced with "Composition" — using different agents/models/roles in a workflow pipeline (e.g., plan with Claude/Opus, implement with Gemini/Flash, review with Claude/Sonnet). `--agent all` default removed; `--agent` now provides a default for tasks without an `agent` field. CLI flags `--model` and `--effort` added. Per-task `agent`, `model`, and `effort` fields added to `TaskDefinition` (all optional). Task-level values take precedence over CLI flags, enabling composition pipelines. `NormalizedTokenUsage` reframed as consistent budget tracking, not cross-agent comparison. Report arrays retained for multi-task directory runs (one entry per task execution, not per agent). AGENTS.md "Cross-Agent Comparison" heading renamed to "Cross-Agent Reference." Per-agent effort control mechanisms documented (Claude: env var, Codex: config flag, Gemini: settings.json).

---

### ~~6. Context window monitoring is a phantom feature~~ — RESOLVED

**Status:** Resolved. Context pressure monitoring is now the **core feature** of rein, not a phantom. Complete redesign:

- **BRIEF.md** reframed: context pressure is the primary operational metric, elevated to its own pillar ("Five Pillars" instead of four). Token budget repositioned as a complementary cost constraint.
- **SESSIONS.md** completely rewritten: Step 4 (MONITOR) is now the defining step. Full protocol documented: pressure zones (Green 0–60%, Yellow 60–80%, Red >80% — configurable globally and per-model), zone actions (Yellow = graceful stop + rein wrap-up, Red = immediate kill + rein wrap-up), per-agent measurement capabilities, degraded mode for agents without mid-run tokens, rein wrap-up protocol, post-kill summary agent, and agent prompt engineering for defense in depth.
- **TOKENS.md** now defines `ContextPressure` and `ZoneConfig` data structures, model context window lookup (config-based with agent-reported override for Claude).
- **AGENTS.md** cross-agent table corrected: Claude and Gemini both support `--output-format stream-json` (not "Single JSON object"). Streaming event types documented per agent. Mid-run token availability: Claude (yes, except extended thinking), Codex (yes, per-turn deltas), Gemini (no — tokens only in final event, OpenTelemetry as future path). Signal behavior documented (SIGINT/SIGTERM per agent).
- **ARCHITECTURE.md** updated: system diagram shows Pressure Monitor subsystem, execution flow includes real-time monitoring loop with zone branching, project structure includes `monitor.py`.
- Multi-turn contradiction removed: context pressure accumulates within a single execution's internal turns, not across multi-turn sessions.
- Threshold scale reconciled: Yellow zone onset (60%) is now in the same order of magnitude as budget WARNING (80%), both representing the acceleration zone where quality/resource risk increases.

---

## Medium

### 7. Turn limit asymmetry across agents

**Documents:** AGENTS.md, SESSIONS.md

Claude's invocation includes `--max-turns 50`. Neither Codex nor Gemini expose an equivalent CLI flag. Gemini has `maxSessionTurns` in settings.json but it is not a CLI parameter the adapter can set per-task.

**Mitigated by #6 resolution:** The context pressure monitor now provides a universal guardrail that bounds execution regardless of turn limits. Rein reads NDJSON/JSONL streams in real-time for Claude and Codex, classifies context pressure into zones, and intervenes with hard kills (Yellow = graceful stop after current turn, Red = immediate SIGTERM/SIGKILL). This makes `--max-turns` a defense-in-depth measure for Claude rather than the sole safeguard. Gemini operates in degraded mode (post-completion monitoring only) but is still bounded by `timeout_seconds`.

**Remaining gap:** The asymmetry is cosmetic for Claude and Codex (context pressure is the real guardrail), but Gemini lacks both mid-run token visibility and a CLI turn limit — it relies solely on `timeout_seconds` and post-completion budget analysis. A `max_turns` field on `TaskDefinition` would still be useful for agents that support it.

**Fix:** Document the asymmetry in the cross-agent table. Consider a `max_turns` field on `TaskDefinition` passed to agents that support it. For Codex, the adapter could count `turn.completed` events in the JSONL stream as a secondary bound.

---

### ~~8. Timeout enforcement is unspecified~~ — RESOLVED

**Status:** Resolved. Timeout enforcement is now fully specified via the shared Subprocess Termination Procedure in ARCHITECTURE.md:

- **Signal sequence:** `SIGINT` for Codex (triggers `TurnAborted`), `SIGTERM` for Claude/Gemini; 5-second grace period; `SIGKILL` if still alive.
- **Partial output:** All complete NDJSON/JSONL lines are captured and parseable. Incomplete final line is discarded. Since issue #6 changed all agents to streaming NDJSON/JSONL output, the original concern about truncated single-JSON blobs no longer applies.
- **Exit codes:** Process `returncode` recorded as-is (typically `-15` SIGTERM, `-9` SIGKILL, `130` Codex SIGINT).
- **Report field:** `termination_reason` added to the report schema with enum values: `completed`, `timed_out`, `context_pressure`, `error`. Replaces the proposed `timed_out: bool` with a richer enum.
- **Shared procedure:** The same termination procedure is used by both context pressure zone actions and wall-clock timeout, with two modes (graceful for yellow zone, immediate for red zone and timeout).

Cross-references: ARCHITECTURE.md (Subprocess Termination Procedure, execution flow step 6), SESSIONS.md (Timeout subsection, Zone Actions, Rein Wrap-Up Protocol), REPORTS.md (`termination_reason` field), TASKS.md (`timeout_seconds` field description), AGENTS.md (Timeout behavior row in cross-agent table).

---

### 9. Gemini JSON fallback is undefined

**Documents:** AGENTS.md

AGENTS.md line 298 states: "The adapter should catch `json.JSONDecodeError` and fall back." But "fall back" is not specified:

- What does `AgentResult` look like when JSON parsing fails?
- Is `NormalizedTokenUsage` zeroed out or treated as an error?
- Is raw stdout preserved?
- Does this count as a failed run?
- Is there a `parse_error` flag in the report?

**Fix:** Define the fallback explicitly: return `AgentResult` with `exit_code` from the process, `result_text` set to raw stdout, `NormalizedTokenUsage` zeroed, and a `parse_error: true` flag. Add `raw_output: str` to `AgentResult` for all agents, populated regardless of parse success.

---

### 10. Execution flow step count and ordering mismatch

**Documents:** ARCHITECTURE.md, SESSIONS.md

ARCHITECTURE.md defines 10 execution steps. SESSIONS.md defines 8 lifecycle steps for the same process. Both place artifact capture before validation:

- ARCHITECTURE.md: step 7 "Capture diff and artifacts" → step 8 "Run validation commands"
- SESSIONS.md: step 5 "CAPTURE" → step 6 "VALIDATE"

This means artifacts created or modified by validation commands (e.g., test output files, coverage reports) are not captured. If validation is meant to be purely observational (exit code only), this is fine but should be stated explicitly. If validation side-effects matter, the ordering needs to change.

ARCHITECTURE.md step 7 says "Capture diff and artifacts before sandbox cleanup" but cleanup is not an explicit step in the 10-step flow — the phrase is misleading in context.

**Fix:** Align both documents to the same step count and numbering. Add a note that validation runs after artifact capture and only exit codes and stdout/stderr are retained from validation. Consider whether a second artifact capture pass after validation is warranted.

---

### 11. Report format internal inconsistency

**Documents:** SESSIONS.md, ARCHITECTURE.md

Two issues:

**Cache efficiency presence.** The standalone budget analysis example (SESSIONS.md line 65) includes `cache_efficiency: { cache_read_tokens, cache_write_tokens }`. The full report format example (SESSIONS.md line 161) omits `cache_efficiency` from the `budget_analysis` block. Either it is part of `budget_analysis` or it is not.

**Report filename vs multi-agent content.** Reports are saved to `results/{task_id}_{YYYYMMDD_HHMMSS}.json`. The report schema has `results[]` and `evaluations[]` as arrays, implying multiple agents per report. But the filename has no agent identifier. If running against all three agents: is it one report with three array entries, or three reports with near-identical timestamps? This is ambiguous.

**Fix:** Make the report examples consistent regarding `cache_efficiency`. Document whether multi-agent runs produce one report or multiple, and adjust the filename pattern accordingly.

---

### 12. Terminology drift — session, run, execution

**Documents:** All

- SESSIONS.md line 9: "A 'session' in rein is a single task execution"
- SESSIONS.md line 107: "Session Continuation" implies a session can span multiple executions
- README.md/ARCHITECTURE.md: `rein run` could mean one task or many (directory scan)
- BRIEF.md line 47: "dispatching one task to one agent at a time"

"Session", "run", and "execution" are used interchangeably with subtly different scopes. The most confusing overlap: a "session" is defined as single-execution, but "session continuation" implies multi-execution sessions.

**Fix:** Define terms once in a glossary (ARCHITECTURE.md or BRIEF.md):

- **Task**: a unit of work defined by a JSON/YAML file
- **Run**: a single invocation of `rein run`, which may dispatch multiple tasks
- **Execution**: one task dispatched to one agent in one sandbox
- **Session**: the agent's conversation context within an execution; may be continued across executions via `--resume`/`--continue`

---

### 13. CLAUDECODE env var conflict is only in AGENTS.md

**Documents:** AGENTS.md, ARCHITECTURE.md

AGENTS.md line 97 states the Claude adapter "must unset `CLAUDECODE` or pass a clean env dict to the subprocess." This requirement appears nowhere else — not in ARCHITECTURE.md's execution isolation section, not in the adapter protocol, not in any cross-cutting concerns section.

A developer running rein from within a Claude Code session (a plausible workflow) would get an opaque failure. The docs also don't define what a "clean env dict" contains — a fully clean env would break `PATH`, `HOME`, `CODEX_API_KEY`, etc.

**Fix:** Move environment variable handling to ARCHITECTURE.md as a cross-cutting concern under Execution Isolation. Specify that adapters copy `os.environ` and delete known conflict variables. Audit Codex and Gemini for analogous env var conflicts.

---

### 14. Prompt length limits — no validation, no stdin fallback

**Documents:** AGENTS.md, TASKS_JSON.md

All three agents receive the prompt as a CLI argument. On Linux, `execve` has a `ARG_MAX` limit (~2 MB, but per-argument limits can be lower). The task schema places no length limit on `prompt`.

AGENTS.md line 196 notes Gemini supports `echo "text" | gemini` (stdin piping). Claude and Codex stdin support is not documented. No adapter uses stdin.

**Fix:** For robustness, prefer stdin piping when the prompt exceeds a conservative threshold (e.g., 100 KB). Verify stdin support for each CLI. At minimum, validate prompt length at task load time and warn or error if it exceeds safe limits.

---

## Low

### 15. `--parallel` flag listed as existing CLI option

ARCHITECTURE.md line 195 lists `--parallel` in the CLI options table with the note "(future)". It reads as an implemented flag. If it is not implemented, listing it in the CLI reference is misleading.

**Fix:** Move to a "Planned" section or add `[not yet implemented]` clearly.

---

### 16. TASKS_YAML.md written in present tense

Despite the "Status: Planned" banner, the document uses present tense throughout and shows working command examples like `rein run -t tasks/example_fizzbuzz.yaml -a claude`.

**Fix:** Use future tense consistently, or add `[planned]` markers to command examples.

---

### 17. Cross-agent comparison table is incomplete

AGENTS.md line 313-315 cross-agent table:

- Claude session resume lists `--resume` but omits `--continue` (different flag, different behavior)
- Codex session resume lists `resume <id>` but omits `resume --last`
- Table entries are abbreviated vs the detailed sections above them

**Fix:** Expand table entries or add footnotes referencing the full flag documentation above.

---

### 18. `asyncio` omitted from dependency table

ARCHITECTURE.md lines 209-214 list dependencies as `click`, `rich`, and stdlib `json`. The entire adapter layer is async using `asyncio.create_subprocess_exec`. While `asyncio` is stdlib, it is architecturally significant and its omission from the dependency table misrepresents the runtime model.

**Fix:** Add `asyncio` to the stdlib row in the dependency table. Consider noting that the project uses an async runtime.

---

### 19. Codex `--full-auto` vs `--yolo` ambiguity

AGENTS.md documents both flags for Codex:

- `--full-auto`: "Auto-approve edits (combines approvals + workspace-write sandbox)"
- `--yolo`: "Remove all safeguards"

The invocation example uses `--full-auto`. The cross-agent table maps Codex auto-approve to `--full-auto`. But `--yolo` is more permissive. Rein should document which one it uses and why.

**Fix:** State explicitly which flag the adapter uses. If `--full-auto` is chosen for safety, document why `--yolo` is not used despite Gemini's adapter using `--yolo`.

---

## Summary

| #  | Issue | Severity | Category |
|----|-------|----------|----------|
| 1  | ~~`total_tokens` always 0~~ | ~~Critical~~ | ~~Design bug~~ — **RESOLVED** |
| 2  | ~~No brownfield sandbox support~~ | ~~Critical~~ | ~~Greenfield/brownfield gap~~ — **RESOLVED** |
| 3  | ~~Codex fails without git~~ | ~~Critical~~ | ~~Adapter edge case~~ — **RESOLVED** |
| 4  | ~~Cache token semantics ambiguous~~ | ~~High~~ | ~~Token normalization~~ — **RESOLVED** |
| 5  | ~~"Not a benchmark tool" contradiction~~ | ~~High~~ | ~~Framing~~ — **RESOLVED** |
| 6  | ~~Context window monitoring is phantom~~ | ~~High~~ | ~~Phantom feature~~ — **RESOLVED** |
| 7  | Turn limit asymmetry across agents | Low | Defense-in-depth gap (Gemini only) |
| 8  | ~~Timeout enforcement unspecified~~ | ~~Medium~~ | ~~Error handling~~ — **RESOLVED** |
| 9  | Gemini JSON fallback undefined | Medium | Error handling |
| 10 | Execution flow ordering mismatch | Medium | Cross-doc inconsistency |
| 11 | Report format internal inconsistency | Medium | Schema |
| 12 | Terminology drift | Medium | Clarity |
| 13 | CLAUDECODE env var buried | Medium | Cross-cutting concern |
| 14 | Prompt length limits | Medium | Edge case |
| 15 | `--parallel` listed as existing | Low | Future-as-present |
| 16 | TASKS_YAML.md present tense | Low | Future-as-present |
| 17 | Cross-agent table incomplete | Low | Documentation gap |
| 18 | `asyncio` omitted from deps | Low | Documentation gap |
| 19 | Codex flag ambiguity | Low | Documentation gap |

### Top 3 Structural Recommendations

1. **Add a `workspace` field to TaskDefinition** to support brownfield projects — worktree from existing repo, copy of existing directory, or tempdir (current default). This is the single biggest gap.

2. **Define error paths** — timeout handling, JSON parse failure, agent startup failure, and runaway execution. The happy path is well-specified; unhappy paths are not.

3. ~~**Resolve cache token semantics**~~ — **RESOLVED.** Claude is exclusive (partitioned), Codex and Gemini are inclusive (subset). Claude adapter now sums all three partitions. See TOKENS.md Cache Semantics.
