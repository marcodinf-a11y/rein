# ADR-002: Completion Promise Signal

## Status

Accepted

## Context

Rein's quality gate is binary — validation commands either all pass (score 1.0) or any fails (score 0.0). This tells us whether the work is correct but not whether the agent believes it finished. These are different signals:

- An agent can produce correct partial work that happens to pass validation (weak validation commands).
- An agent can believe it's done while validation fails (hallucinated success / reward hacking).
- An agent killed by context pressure or timeout never had a chance to finish — the binary gate doesn't distinguish "didn't finish" from "finished incorrectly."

Ralph Wiggum uses a "completion promise" string the agent must emit to signal genuine completion. Ralph Orchestrator uses `ralph emit LOOP_COMPLETE` with required-events validation. Both demonstrate that an agent's self-assessment, cross-referenced against objective validation, produces a more informative signal than either alone.

## Decision

The agent writes a marker file `.rein/complete` in the sandbox when it believes the task is complete. Rein checks for this file during the VALIDATE step (after agent exit, before sandbox cleanup) and cross-references it against validation command results to produce a six-outcome confidence classification:

| Promise Filed | Validation Passed | Outcome | Interpretation |
|--------------|-------------------|---------|---------------|
| Yes | Yes | **Confident** | Agent believes it's done, and it is. |
| No | Yes | **Suspicious** | Validation passes but agent didn't signal completion. Weak validation or agent ran out of context. |
| Yes | No | **Overconfident** | Agent claims done but validation fails. Hallucination or reward hacking. |
| No | No | **Incomplete** | Agent didn't finish and validation confirms it. Expected for context-pressure kills and timeouts. |
| Yes | null | **Unverified** | Agent claims done but no validation commands configured. Promise cannot be verified. |
| No | null | **Unevaluated** | No completion promise and no validation. No quality signal available. |

`validation_passed` is `null` when `validation_commands` is empty (no validation ran — distinct from pass or fail). See FR-067.

Specific decisions:

- **Marker path is hardcoded** to `.rein/complete`. No per-task configuration. The `.rein/` directory is rein's namespace.
- **File existence is the boolean signal.** `completion_promise=true` if the file exists, regardless of content. Empty file counts.
- **File content is captured** as `completion_summary` in the report (the agent's self-assessment). Empty string if the file is empty.
- **The prompt suffix instructs the agent** to verify its work before writing the marker. This is assembled by rein's prompt layer (see PROMPTS.md §5) and can be omitted by the operator.
- **Backward compatible.** Tasks without the prompt suffix never produce a marker. `completion_promise=false`, confidence determined by validation alone: pass → `suspicious`, fail → `incomplete`, no validation → `unevaluated`.
- **No special handling for context-pressure kills or timeouts.** The six-outcome matrix handles them naturally — the agent didn't write the marker, so it classifies as `suspicious`, `incomplete`, or `unevaluated` depending on validation.

Three new fields in the evaluation section of the JSON report:

- `completion_promise`: `boolean` — whether the marker file existed
- `completion_confidence`: `"confident" | "suspicious" | "overconfident" | "incomplete" | "unverified" | "unevaluated"`
- `completion_summary`: `string | null` — contents of the marker file, or `null` if absent

## Consequences

**Positive:**

- Adds a confidence dimension to the binary quality gate without replacing it.
- "Overconfident" failures directly detect hallucinated success — a signal the binary gate cannot produce.
- "Suspicious" passes flag weak validation commands, improving task definition quality over time.
- Near-zero implementation cost: one `os.path.exists()` check, one file read, one cross-reference.

**Negative:**

- Relies on the agent following prompt instructions. An agent that ignores the instruction or writes the marker prematurely degrades the signal (but doesn't break anything — validation still runs independently).
- Adds three fields to every evaluation entry. Operators who don't use the prompt suffix will see `completion_promise=false` on every run.

**Neutral:**

- The marker file is captured before sandbox cleanup, so it does not survive beyond the report.
- No interaction with context pressure monitoring — the completion promise is evaluated post-execution only.
