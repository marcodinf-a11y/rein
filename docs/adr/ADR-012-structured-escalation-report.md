# ADR-012: Structured Escalation Report on Terminal Failure

## Status

Accepted

## Context

When a task exhausts `max_rounds` with a final verdict of `"fail"`, rein records the failure in the report — but the report is designed for machines and trend analysis, not for a human picking up the work. The raw data is all there (signal results, validation output, diffs per round), but a developer looking at a failed task needs to answer three questions quickly:

1. What did the agent try?
2. What specifically failed?
3. Where is the work so I can continue from it?

The existing report answers these questions, but spread across nested `rounds[]` entries that require manual traversal. A structured escalation report aggregates the failure narrative into one self-contained block.

This pattern is adopted from Stripe's Minions system, where failed tasks escalate to humans with context about what was attempted and what failed, preserving the agent's branch and changes rather than discarding them.

Additional patterns from research deep dives informed the schema:
- **Principal Skinner** (circuit breakers): distinguishing *why* escalation happened — financial limits (max rounds, budget) vs. behavioral limits (context pressure, stagnation)
- **Ralph Orchestrator** (completion promise + stagnation detection): cross-referencing the agent's self-assessment with validation results to classify failure confidence
- **Ralph Orchestrator** (knowledge persistence): surfacing operational facts the agent discovered during its failed run as context for the human

### Options considered

**Option A — Mechanical summary only.** Rein generates the escalation report deterministically from signal data. Template-driven: "Round 1: tests failed (2/5), review requested changes. Round 2: tests failed (1/5), review approved." Zero additional token cost. Zero failure modes.

**Option B — LLM-generated summary only.** A fresh summary agent reads the round history and produces a semantic narrative. Higher quality ("the agent tried adding a null guard but missed that `parse_input` is also called from `validate_token`"), but costs ~10-15k tokens on top of an already-failed run, and can itself fail or hallucinate.

**Option C — Mechanical default with opt-in LLM enrichment.** Rein always produces a mechanical summary. If `[escalation] summary_agent = true` in `rein.toml`, a summary agent enriches `approach_summary` and `diagnostic_summary` fields. On summary agent failure, the mechanical version is preserved.

## Decision

**Option C — mechanical default with opt-in LLM enrichment.**

### Escalation report schema

The `escalation_report` field is a sibling to `final_verdict` in the top-level report JSON. It is populated only when `final_verdict == "fail"`. Otherwise `null`.

```json
{
    "escalation_report": {
        "task_summary": "Refactor auth module to use JWT",
        "rounds_attempted": 4,
        "max_rounds": 4,

        "round_history": [
            {
                "round_number": 1,
                "approach_summary": "Round 1: tests failed (2/5), review requested changes.",
                "termination_reason": "completed",
                "failing_signals": [
                    {
                        "name": "tests",
                        "message": "2 of 5 tests failed (test_edge_case, test_empty_input)",
                        "output_excerpt": "AssertionError: parse_input(None) raised TypeError"
                    },
                    {
                        "name": "review",
                        "message": "Requested changes: missing null check",
                        "output_excerpt": null
                    }
                ],
                "passing_signals": ["build", "lint", "context_pressure", "diff_size"]
            },
            {
                "round_number": 2,
                "approach_summary": "Round 2: tests failed (1/5). Progress: 1 failure resolved.",
                "termination_reason": "completed",
                "failing_signals": [
                    {
                        "name": "tests",
                        "message": "1 of 5 tests failed (test_edge_case)",
                        "output_excerpt": "AssertionError: expected 'admin' got 'user' for role escalation"
                    }
                ],
                "passing_signals": ["build", "lint", "review", "context_pressure", "diff_size"]
            }
        ],

        "diagnostic_summary": "Agent resolved 1 of 2 test failures across 2 rounds. Remaining failure: test_edge_case (role escalation logic). Progress: improved.",

        "escalation_trigger": "max_rounds_exhausted",

        "completion_confidence": "overconfident",

        "learnings_snapshot": [
            "- test: `pytest tests/ --test-threads=1` (DB tests not parallelizable)",
            "- quirk: auth module uses custom JWT, not the jsonwebtoken crate"
        ],

        "preserved_state": {
            "branch": "rein/refactor-auth-001-20260306",
            "last_commit": "a1b2c3d",
            "diff_stat": "5 files changed, 125 insertions(+), 30 deletions(-)"
        },

        "summary_agent_used": false
    }
}
```

### Field descriptions

| Field | Description |
|-------|-------------|
| `task_summary` | Task name from the task definition. Quick identifier. |
| `rounds_attempted` | How many rounds ran before exhaustion. |
| `max_rounds` | The configured limit. |
| `round_history[]` | Per-round failure narrative. |
| `round_history[].approach_summary` | What happened this round. Mechanical by default; LLM-enriched if summary agent enabled. |
| `round_history[].termination_reason` | How the round ended: `completed`, `context_pressure`, `timed_out`, `error`. |
| `round_history[].failing_signals[]` | Signals that failed this round, with their message and an output excerpt. |
| `round_history[].failing_signals[].message` | From `SignalResult.message` — already framework-agnostic. |
| `round_history[].failing_signals[].output_excerpt` | Last 30 lines of relevant output after the first failure marker, or last 30 lines if no marker found. `null` for signals without command output (e.g., review, context_pressure). Full output remains in `rounds[].evaluation.validation_output`. |
| `round_history[].passing_signals` | List of signal names that passed — keeps the report scannable without listing full details for successes. |
| `diagnostic_summary` | Overall assessment. Mechanical: "Resolved X of Y failures across N rounds. Remaining: {list}. Progress: improved/stagnated/regressed." LLM-enriched if summary agent enabled. |
| `escalation_trigger` | Why escalation happened. One of: `max_rounds_exhausted` (normal case — all rounds ran, all failed), `context_pressure` (every round was killed by zone pressure), `timed_out` (every round hit the wall-clock limit), `error` (agent crashed every round). Derived from `termination_reason` of the final round, but surfaced explicitly so the operator doesn't have to dig through round history. Informed by Principal Skinner's circuit breaker taxonomy (financial vs. behavioral limits). |
| `completion_confidence` | From the final round's completion promise cross-reference ([ADR-002](docs/adr/ADR-002-completion-promise-signal.md)). One of: `overconfident` (agent claimed done but validation failed — the code is wrong in a way the agent can't see), `incomplete` (agent didn't claim done — it knew it wasn't finished), `unverified` (agent claimed done but no validation commands configured — promise cannot be verified). Values that should not appear in escalation: `suspicious` / `confident` (validation passed → verdict would be pass, not fail), `unevaluated` (no promise + no validation → gate didn't fail for validation reasons). Informed by Ralph Orchestrator's completion promise + required-events validation pattern. |
| `learnings_snapshot` | New entries from LEARNINGS.md that were extracted from the sandbox during this session ([ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)). These are operational facts the agent discovered during its failed run (build commands, test quirks, environment requirements) that provide context for the human picking up the work. Empty list if no learnings were extracted. Informed by Ralph Orchestrator's knowledge persistence pattern. |
| `preserved_state.branch` | Git branch with the agent's work, if available. `null` for tempdir workspaces where the sandbox is deleted. |
| `preserved_state.last_commit` | Short SHA of the last commit in the branch. |
| `preserved_state.diff_stat` | `git diff --stat` summary of total changes. |
| `summary_agent_used` | Whether the LLM summary agent was used to enrich summaries. |

### Mechanical summary generation

The mechanical summary requires no LLM. It is deterministic and computed from signal data:

**`approach_summary` template:**
```
Round {n}: {failing_signal_names} failed ({counts}). {progress_note}
```

Where `progress_note` compares against the previous round:
- First round: omitted
- Fewer failures than previous: "Progress: {N} failure(s) resolved."
- Same failures: "No progress from previous round."
- More failures: "Regression: {N} new failure(s)."

**`diagnostic_summary` template:**
```
Agent resolved {resolved} of {total_initial} failures across {N} rounds.
Remaining: {failing_signal_list}. Progress: {improved|stagnated|regressed}.
```

Progress classification:
- `improved`: final round has fewer failing signals than round 1
- `stagnated`: final round has same number of failing signals as round 1
- `regressed`: final round has more failing signals than round 1

### `output_excerpt` extraction

The `output_excerpt` field provides just enough failure context to be actionable without duplicating the full validation output.

Extraction logic:
1. Scan `validation_output` for common failure markers: `FAILED`, `FAIL:`, `Error:`, `●`, `✗`, `AssertionError`, `assert`
2. If a marker is found: extract up to 30 lines starting from the first marker
3. If no marker is found: extract the last 30 lines of output
4. If output is empty or signal has no command output: `null`

The full output is always available in `rounds[].evaluation.validation_output` for deeper investigation.

### Opt-in LLM summary enrichment

When enabled via `rein.toml`, a summary agent enriches the `approach_summary` and `diagnostic_summary` fields with semantic context.

```toml
[escalation]
summary_agent = false          # default: mechanical only
summary_model = "haiku"        # cheap/fast model
summary_token_budget = 10000
summary_timeout_seconds = 60
```

The summary agent receives:
- The task prompt (what was asked)
- Per-round git diff stats
- Per-round failing signal results with output excerpts
- The mechanical summary (as a baseline to improve upon)

On summary agent failure (timeout, parse error, hallucination detection), rein falls back to the mechanical summary and logs a warning. The `summary_agent_used` field in the report indicates which path was taken.

The summary agent runs in a fresh context (zero pressure), uses the configured model (default: haiku for cost efficiency), and is budget-limited to prevent runaway spend on failure reporting.

### When the escalation report is generated

The escalation report is generated in the REPORT step (step 8 of the session lifecycle), after learnings extraction (step 7) and before cleanup (step 9). It only fires when `final_verdict == "fail"`.

```
After final verdict == "fail":

1. Collect round history from rounds[] data
2. For each round: extract failing/passing signals, compute approach_summary
3. Compute diagnostic_summary (progress classification)
4. Derive escalation_trigger from final round's termination_reason
5. Read completion_confidence from final round's evaluation
6. Snapshot learnings extracted in step 7 of session lifecycle (ADR-011)
7. Capture preserved_state (branch, commit, diff_stat)
8. If summary_agent enabled: dispatch summary agent for enrichment
9. On summary agent failure: fall back to mechanical, log warning
10. Attach escalation_report to report JSON
```

### Code location

In `runner.py` (orchestrator), after the round loop and learnings extraction:

```python
# After final verdict, before report serialization
if final_verdict == "fail":
    escalation = escalation.build_report(rounds, task, config)
    report["escalation_report"] = escalation
```

An `escalation.py` module provides:
- `build_report(rounds, task, config) -> dict` — assembles the escalation report
- `mechanical_summary(rounds) -> str` — generates approach/diagnostic summaries
- `extract_output_excerpt(validation_output, max_lines=30) -> str | None` — extracts failure context
- `classify_progress(rounds) -> str` — returns `"improved"`, `"stagnated"`, or `"regressed"`

## Rationale: Why Option C Over A and B

**Option A (mechanical only) is always correct but sometimes insufficient.** A template saying "1 of 5 tests failed" is barely better than scanning the raw signals. The human still has to read the diff and test output to understand *why*. For simple failures this is fine; for subtle bugs, a semantic summary saves significant time.

**Option B (LLM only) adds cost and failure modes to the failure path.** The escalation report fires when the task has already consumed `max_rounds × token_budget` tokens and produced nothing usable. Adding another LLM call on top of that — one that can itself fail — is questionable. The developer is already frustrated; a broken error report compounds the problem.

**Option C gives both guarantees.** The mechanical summary always works, costs nothing, and has zero failure modes. The LLM enrichment is opt-in, uses a cheap model, and falls back gracefully. Users who want richer failure context pay for it explicitly. Users who don't get a useful-enough report for free.

The `summary_agent_used` field in the report makes it clear which path produced the summaries, so operators can evaluate whether the LLM enrichment is worth the cost for their workflow.

## Consequences

**Positive:**

- Failed tasks produce actionable context for the human picking them up — not just "task failed."
- Mechanical default means zero additional cost or failure risk on the failure path.
- Opt-in LLM enrichment available for users who want semantic summaries.
- Progress classification (improved/stagnated/regressed) across rounds is computed automatically.
- `preserved_state` tells the developer exactly where to find the agent's work.

**Negative:**

- The `output_excerpt` extraction uses heuristic failure markers. Unusual test frameworks may not match, falling back to "last 30 lines" which may not contain the failure. Mitigation: full output is always in `rounds[].evaluation.validation_output`.
- The summary agent (when enabled) adds ~10k tokens of cost to failed runs. Mitigation: opt-in, cheap model default, budget-capped.
- The mechanical `approach_summary` is formulaic and doesn't capture semantic nuance. Mitigation: this is the trade-off for zero cost and zero failure modes; LLM enrichment is available.

**Neutral:**

- The escalation report is a reporting enhancement, not an architectural change. It reads data that already flows through the round/signal model.
- The `escalation.py` module follows the same pattern as `learnings.py` — a focused module with a clear interface called from `runner.py`.
- The `[escalation]` config section in `rein.toml` follows the same pattern as other opt-in features (review agent).
