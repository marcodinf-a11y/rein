# Agentic Harness — Inconsistency & Gap Analysis

Cross-document review of all seven specification files, with focus on greenfield vs brownfield support, internal contradictions, and undocumented edge cases.

---

## Critical

### 1. `total_tokens` is always 0 as documented

**Documents:** ARCHITECTURE.md, AGENTS.md

The frozen dataclass defines `total_tokens: int = 0` with a comment "Computed: input + output" (ARCHITECTURE.md line 113). But:

- No `__post_init__` is shown that would compute the value
- Frozen dataclasses do not allow attribute setting after initialization
- All three adapter functions (`_normalize_claude`, `_parse_codex_jsonl`, `_normalize_gemini`) construct `NormalizedTokenUsage` without setting `total_tokens`
- Budget analysis at ARCHITECTURE.md line 151 uses `usage.total_tokens` directly

Every adapter as documented will produce `total_tokens = 0`. Budget utilization will always report 0%.

**Fix:** Add `__post_init__` using `object.__setattr__`:

```python
def __post_init__(self):
    object.__setattr__(self, 'total_tokens', self.input_tokens + self.output_tokens)
```

Update all three adapter code samples to either rely on `__post_init__` or explicitly pass `total_tokens`.

---

### 2. Sandbox model has no brownfield story

**Documents:** BRIEF.md, ARCHITECTURE.md, TASKS_JSON.md, SESSIONS.md

BRIEF.md line 29 says "temp directory, worktree" and ARCHITECTURE.md line 41 describes `sandbox.py` as "Execution isolation (worktrees/tmp dirs)". But every concrete specification assumes a fresh temporary directory:

- ARCHITECTURE.md line 62: "Creates a temporary directory for each run"
- Execution flow step 2: "Create isolated sandbox (temp directory)"
- Session lifecycle step 2: "Create temp directory"
- All task examples use `git init` in `setup_commands` — greenfield only

There is no mechanism for brownfield use:

- No `workspace` or `source_dir` field in the task schema to point at an existing codebase
- No documented worktree creation logic anywhere
- The `files` field requires inlining all content as JSON strings — does not scale beyond trivial seed files
- No explanation of how diff baselines are established against an existing project
- Validation commands that depend on broader project context (integration tests, shared modules) have no access path

The BRIEF.md states this tool is for "actual development work" on existing codebases, but the entire execution model is greenfield-only.

**Fix:** Add a `workspace` field to `TaskDefinition`:

```python
@dataclass(frozen=True)
class WorkspaceConfig:
    type: str          # "tempdir" | "worktree" | "copy"
    source: str = ""   # Path to existing repo/directory (required for worktree/copy)
```

Document worktree behavior in ARCHITECTURE.md under Execution Isolation. Define how diff baselines are captured (commit hash at sandbox creation time). Consider a `files_from` field or glob pattern as an alternative to inlining file contents.

---

### 3. Codex will fail in bare sandboxes

**Documents:** AGENTS.md, ARCHITECTURE.md

The documented Codex invocation is:

```bash
codex exec --json --full-auto "PROMPT"
```

This does not include `--skip-git-repo-check`. AGENTS.md line 181 states Codex "Requires a Git repository by default." The sandbox is a temporary directory that only has `.git` if the task author includes `git init` in `setup_commands`. If they don't, Codex fails.

Additionally, the JSONL parser (`_parse_codex_jsonl`) only handles `turn.completed` and `item.completed` events. The `error` and `turn.failed` event types listed at AGENTS.md line 135 are not parsed. A git-related failure would produce an unhandled event type, resulting in zero tokens and an empty last_message with no error context.

**Fix:** The Codex adapter should always pass `--skip-git-repo-check`, or check for `.git` in the sandbox and inject the flag conditionally. The JSONL parser must handle `error` and `turn.failed` events.

---

## High

### 4. Cache token semantics are ambiguous — budget math may be wrong

**Documents:** AGENTS.md, ARCHITECTURE.md

ARCHITECTURE.md line 118 states: "`total_tokens` is always `input_tokens + output_tokens`." ARCHITECTURE.md line 120 states: "Cache tokens are tracked but not added to the total."

But the relationship between `input_tokens` and cache tokens is unclear per agent:

**Claude Code:** The JSON example shows `input_tokens: 3` alongside `cache_read_input_tokens: 13829`. If these are separate (additive), total input consumption was 16,832. But the normalization maps `input_tokens` to just 3. The budget would be `3 + 21 = 24 tokens` — absurdly low for a real invocation. Is Claude's `input_tokens` the non-cached portion only, or the full input count?

**Gemini CLI:** `prompt: 24939` and `cached: 21263`. If `cached` is a subset of `prompt`, net new input is 3,676. If they are additive, total input is 46,202. The normalization maps `input_tokens` to `prompt` (24,939) without clarification.

This ambiguity means the budget calculation could be off by an order of magnitude depending on interpretation. The docs must specify whether each agent's "input" field is inclusive or exclusive of cached tokens.

**Fix:** Research the actual semantics of each agent's token fields. Document explicitly:

- Claude: `input_tokens` is [inclusive/exclusive] of `cache_read_input_tokens`
- Codex: `input_tokens` is [inclusive/exclusive] of `cached_input_tokens`
- Gemini: `prompt` is [inclusive/exclusive] of `cached`

Update the normalization rules accordingly. If fields are exclusive, the budget formula may need adjustment.

---

### 5. "Not a benchmarking tool" contradicts its own design

**Documents:** BRIEF.md, ARCHITECTURE.md, README.md

BRIEF.md line 8: "This is not a benchmarking or comparison tool."

But:

- BRIEF.md line 59: "Comparison — run the same task against multiple agents, compare results"
- ARCHITECTURE.md line 234: "`NormalizedTokenUsage` allows cross-agent comparison"
- ARCHITECTURE.md line 191: CLI default is `--agent all`, running against all agents
- README.md line 32: Quick start example without `-a` flag implies all-agent default
- The entire `NormalizedTokenUsage` model and the report format with `results[]` arrays are designed for cross-agent comparison

The tool's architecture, defaults, and planned features are those of a comparison tool. The disclaimer contradicts the design.

**Fix:** Soften the language. Replace "This is not a benchmarking or comparison tool" with something like: "This is a development workflow tool, not a synthetic benchmarking suite. Cross-agent comparison is a secondary capability, not the primary purpose."

---

### 6. Context window monitoring is a phantom feature

**Documents:** SESSIONS.md vs all others

SESSIONS.md lines 79-89 define context window monitoring with Green/Yellow/Red thresholds (0-40%, 41-50%, 51-100%). But:

- No other document references these thresholds
- No enum or data structure is defined for them (unlike `BudgetStatus` which has `WITHIN`/`WARNING`/`EXCEEDED`)
- Only Claude Code reports `contextWindow` size in its output; Codex and Gemini do not
- The threshold bands are inconsistent with budget thresholds: Red starts at 51% context usage vs Warning at 80% budget usage — a completely different scale with no rationale
- SESSIONS.md line 89 says this is "most relevant for multi-turn sessions" but line 9 defines a session as "a single task execution" and says "The harness does not maintain multi-turn conversations with agents"

This feature has no implementation path, no data source for two of three agents, and is internally contradictory with the session model.

**Fix:** Either remove the context window monitoring section entirely, or:

1. Move it to a "Future" section
2. Define a data structure for it
3. Document which agents support it
4. Reconcile the threshold scale with budget thresholds
5. Clarify when it applies (multi-turn continuation only)

---

## Medium

### 7. No turn limit for Codex and Gemini — runaway execution risk

**Documents:** AGENTS.md

Claude's invocation includes `--max-turns 50`. Neither Codex nor Gemini have an equivalent flag documented. The CLI flag tables (AGENTS.md lines 113-122 for Codex, lines 199-210 for Gemini) list no turn/iteration limit.

The token budget is a reporting metric, not a hard stop — SESSIONS.md confirms the harness does not intervene mid-run. The only guardrail is `timeout_seconds` (default 300). Five minutes of unconstrained agentic execution can consume far more than the 70k budget.

**Fix:** Investigate whether Codex and Gemini CLIs have undocumented turn limit flags. If not, document the asymmetry explicitly. Consider adding a `max_turns` field to `TaskDefinition` that is passed to agents that support it. For Codex, the adapter could monitor `turn.completed` events in the JSONL stream and terminate the subprocess if a threshold is exceeded.

---

### 8. Timeout enforcement is unspecified

**Documents:** TASKS_JSON.md, ARCHITECTURE.md

`timeout_seconds` exists in the task schema (default 300) but the documentation never specifies:

- The enforcement mechanism (`asyncio.wait_for`? Manual timer with `process.kill`?)
- The signal sequence (SIGTERM first, then SIGKILL? Or immediate SIGKILL?)
- Whether partial output is captured on timeout
- What `exit_code` is reported (typically -9 for SIGKILL, -15 for SIGTERM)
- Whether `timed_out` appears anywhere in the report format

For Codex JSONL, partial output is recoverable — every complete line up to the kill point can be parsed. For Claude and Gemini single-JSON output, a timeout almost certainly means incomplete JSON, which collapses to the Gemini JSON fallback problem (issue #9).

**Fix:** Document the timeout sequence: (a) send SIGTERM, (b) wait 5 seconds, (c) send SIGKILL if still alive, (d) drain remaining stdout/stderr. Add `timed_out: bool` to the report. Define partial output handling per agent.

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

- SESSIONS.md line 9: "A 'session' in the harness is a single task execution"
- SESSIONS.md line 107: "Session Continuation" implies a session can span multiple executions
- README.md/ARCHITECTURE.md: `harness run` could mean one task or many (directory scan)
- BRIEF.md line 47: "dispatching one task to one agent at a time"

"Session", "run", and "execution" are used interchangeably with subtly different scopes. The most confusing overlap: a "session" is defined as single-execution, but "session continuation" implies multi-execution sessions.

**Fix:** Define terms once in a glossary (ARCHITECTURE.md or BRIEF.md):

- **Task**: a unit of work defined by a JSON/YAML file
- **Run**: a single invocation of `harness run`, which may dispatch multiple tasks
- **Execution**: one task dispatched to one agent in one sandbox
- **Session**: the agent's conversation context within an execution; may be continued across executions via `--resume`/`--continue`

---

### 13. CLAUDECODE env var conflict is only in AGENTS.md

**Documents:** AGENTS.md, ARCHITECTURE.md

AGENTS.md line 97 states the Claude adapter "must unset `CLAUDECODE` or pass a clean env dict to the subprocess." This requirement appears nowhere else — not in ARCHITECTURE.md's execution isolation section, not in the adapter protocol, not in any cross-cutting concerns section.

A developer running the harness from within a Claude Code session (a plausible workflow) would get an opaque failure. The docs also don't define what a "clean env dict" contains — a fully clean env would break `PATH`, `HOME`, `CODEX_API_KEY`, etc.

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

Despite the "Status: Planned" banner, the document uses present tense throughout and shows working command examples like `harness run -t tasks/example_fizzbuzz.yaml -a claude`.

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

The invocation example uses `--full-auto`. The cross-agent table maps Codex auto-approve to `--full-auto`. But `--yolo` is more permissive. The harness should document which one it uses and why.

**Fix:** State explicitly which flag the adapter uses. If `--full-auto` is chosen for safety, document why `--yolo` is not used despite Gemini's adapter using `--yolo`.

---

## Summary

| #  | Issue | Severity | Category |
|----|-------|----------|----------|
| 1  | `total_tokens` always 0 | Critical | Design bug |
| 2  | No brownfield sandbox support | Critical | Greenfield/brownfield gap |
| 3  | Codex fails without git | Critical | Adapter edge case |
| 4  | Cache token semantics ambiguous | High | Token normalization |
| 5  | "Not a benchmark tool" contradiction | High | Framing |
| 6  | Context window monitoring is phantom | High | Phantom feature |
| 7  | No turn limit for Codex/Gemini | Medium | Runaway execution |
| 8  | Timeout enforcement unspecified | Medium | Error handling |
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

3. **Resolve cache token semantics** — research whether each agent's input token count is inclusive or exclusive of cached tokens, and update normalization rules accordingly. Budget calculations depend on this.
