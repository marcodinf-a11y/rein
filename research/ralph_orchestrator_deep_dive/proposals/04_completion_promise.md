# Proposal: Completion Promise Signal

**Target:** TASKS.md (quality gate enhancement)
**Priority:** Now
**Effort:** Low

---

## Proposed Enhancement: Completion Promise

The following describes a proposed enhancement to the quality gate evaluation, documented in TASKS.md and REPORTS.md.

---

### Completion Promise

A completion promise is a structured signal the agent writes to indicate "I believe I am done." Rein cross-references this signal against `validation_commands` results to produce a three-outcome confidence classification.

#### How It Works

1. The task prompt instructs the agent to write a marker file when it believes the task is complete:

```
When you are confident the task is complete and all requirements are met,
create a file called .rein/complete with a brief summary of what you did.
```

2. After the agent exits, rein checks for the marker file (`.rein/complete`) in the sandbox.

3. Rein cross-references the promise against validation results:

| Promise Filed | Validation Passed | Outcome | Interpretation |
|--------------|-------------------|---------|---------------|
| Yes | Yes | **Confident** | Agent believes it's done, and it is. High confidence. |
| No | Yes | **Suspicious** | Validation passes but the agent didn't signal completion. May be accidental pass, incomplete work, or agent ran out of context before finishing. |
| Yes | No | **Overconfident** | Agent claims completion but validation fails. The agent misjudged its own work — possible hallucination or reward hacking. |
| No | No | **Incomplete** | Agent didn't finish and validation confirms it. Expected for context-pressure kills and timeouts. |

#### Report Schema Addition

```json
{
    "evaluations": [
        {
            "agent_name": "claude-code",
            "task_id": "refactor-auth-001",
            "validation_passed": true,
            "score": 1.0,
            "completion_promise": true,
            "completion_confidence": "confident",
            "completion_summary": "Refactored auth.py to use PyJWT. All 12 tests pass."
        }
    ]
}
```

New fields:
- `completion_promise`: `boolean` — whether the agent wrote the marker file
- `completion_confidence`: `"confident" | "suspicious" | "overconfident" | "incomplete"` — cross-referenced outcome
- `completion_summary`: `string | null` — contents of the marker file (agent's self-assessment)

#### Implementation

- **Marker file location:** `.rein/complete` in the sandbox directory
- **Detection:** `os.path.exists(sandbox / ".rein" / "complete")` after agent exit, before sandbox cleanup
- **Prompt injection:** Add the completion promise instruction to the task prompt via a standard suffix. The operator can omit or customize the instruction.
- **Backward compatible:** If no marker file exists, `completion_promise=false` and `completion_confidence` is determined by validation alone (`"suspicious"` if pass, `"incomplete"` if fail)

---

## Rationale

Ralph-orchestrator uses `ralph emit LOOP_COMPLETE` as a JSONL event that the agent emits during execution. The event loop checks for this event and validates it against `required_events` (e.g., `["build.done", "test.pass"]`) before accepting completion. If required events haven't been seen, completion is rejected and the loop continues.

Rein adaptation is simpler — a marker file instead of a CLI event — because rein runs agents as black-box subprocesses. The agent cannot call `rein emit`; it can only write files. A marker file achieves the same signal: the agent declares "I'm done" in a structured, detectable way.

The four-outcome classification adds a **confidence dimension** that the current binary quality gate lacks. Currently, rein knows only pass/fail. With the completion promise:
- **Suspicious passes** flag runs that might be accidentally passing (e.g., validation commands too weak)
- **Overconfident failures** flag agents that hallucinate success — a direct signal for reward hacking

This is identified as "Now" priority in both the [Ralph Wiggum synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) §7.2 and §8 because it requires no new infrastructure — just a file existence check and a prompt suffix.

## Source References

- [Ralph Wiggum Synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) — Section 7.2 (completion promise pattern)
- [Iteration Control](../03_iteration_control.md) — Section 2 (ralph's LOOP_COMPLETE event, required_events validation)
- TASKS.md — current quality gate (validation_commands, binary scoring)
- REPORTS.md — current evaluation schema
