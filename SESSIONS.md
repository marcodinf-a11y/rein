# Rein — Session Management & Context Pressure Monitoring

How context pressure is monitored in real-time, when rein intervenes, and how work is preserved.

## Why Context Pressure Is the Core Metric

AI coding agents degrade as conversations grow. Research shows this degradation is **continuous and accelerating** — there is no safe plateau, and newer models with larger context windows exhibit the same patterns as older ones ([Chroma Research: Context Rot](https://research.trychroma.com/context-rot)). More context means worse reasoning, more hallucination, and more instruction-following failures.

Rein treats **context pressure** — the ratio of consumed tokens to the model's context window — as its primary operational metric. Every intervention decision (continue, wrap up, kill) is driven by context pressure.

A "session" in rein is a single task execution: one task dispatched to one agent in one sandbox. Rein does not maintain multi-turn conversations with agents — each run is independent by default. Within a single execution, the agent may have many internal turns (tool calls, reasoning steps), and it is across these turns that context pressure accumulates. Session continuation (multi-turn across executions) is supported per-agent but is not the default workflow.

## Session Lifecycle

```
1. LOAD         Load task definition from JSON
                 │
2. SANDBOX      Create sandbox per workspace.type:
                   tempdir  → empty temp directory
                   worktree → git worktree from source repo
                   copy     → copy source tree into temp directory
                 Write seed files, run setup commands
                 │
3. INVOKE       Start agent CLI as subprocess (stream-json mode)
                 cwd = sandbox directory
                 │
4. MONITOR      ← CORE LOOP — real-time context pressure monitoring
                 Parse NDJSON/JSONL stream line by line
                 Track cumulative token usage per turn
                 Compute context pressure (tokens / context window)
                 Classify zone: green → continue
                                yellow → graceful stop (see Zone Actions)
                                red    → immediate kill (see Zone Actions)
                 Concurrently: wall-clock timer against timeout_seconds
                               timeout → immediate kill (see Timeout)
                 │
5. CAPTURE      Parse final/partial output from stream buffer
                 Normalize token usage
                 Capture diff and artifacts
                 │
6. VALIDATE     Run validation commands in sandbox
                 Score: all pass → 1.0, any fail → 0.0
                 │
7. EXTRACT      Extract learnings from sandbox LEARNINGS.md
                 Diff against .rein/LEARNINGS.md in project root
                 Validate new entries, merge back (see ADR-011)
                 │
8. REPORT       Generate structured JSON report
                 Save to results/{task_id}_{timestamp}.json
                 │
9. CLEANUP      Remove sandbox directory
                 (Artifacts and learnings already captured)
```

Step 4 (MONITOR) is the defining step of rein. It runs concurrently with the agent — reading the agent's output stream in real-time and computing context pressure after each turn completes. See [Context Pressure Monitoring](#context-pressure-monitoring) below.

This lifecycle corresponds to the execution flow defined in [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Context Pressure Monitoring

### Concept

Context pressure measures how much of the model's context window has been consumed. As pressure increases, model quality degrades — accelerating, not linear. Rein monitors this in real-time and acts on configurable thresholds.

```
Context Pressure = estimated_tokens_used / model_context_window × 100
```

### Pressure Zones

| Zone | Default Range | Meaning | Rein Action |
|---|---|---|---|
| **Green** | 0–60% | Safe operating range. Degradation present but shallow. | Continue execution. |
| **Yellow** | 60–80% | Acceleration zone. Quality degrading noticeably. | **Graceful stop.** Wait for current turn to complete, then kill subprocess. Rein wraps up. |
| **Red** | >80% | Steep degradation. High hallucination and instruction-following failure risk. | **Immediate kill.** Rein wraps up with partial data. |

### Configurable Thresholds

Zone boundaries are configurable globally and per-model. When a model has a large context window but quality degrades early (e.g., a 1M-token window where quality drops steeply at 400k), shrink the effective boundaries.

```yaml
context_pressure:
  green_max_pct: 60
  yellow_max_pct: 80
  # Above yellow_max = red

  overrides:
    # Model with large window but early degradation
    "some-large-window-model":
      green_max_pct: 40
      yellow_max_pct: 60
```

**Buffer zone warning:** The gap between `green_max_pct` and `yellow_max_pct` is the "landing strip" — the agent's remaining context room to finish its current turn before rein intervenes. If this gap is too small, the agent may push from green into red within a single turn, bypassing yellow entirely. A gap of at least 15–20% of the context window is recommended for typical coding tasks. Rein does not enforce a minimum gap — this is the operator's responsibility to configure sensibly based on task complexity and model behavior.

### Per-Agent Measurement Capabilities

Not all agents provide the same level of mid-run token visibility:

| Agent | Mid-Run Monitoring | Mechanism | Granularity |
|---|---|---|---|
| **Claude Code** | Yes | `--output-format stream-json` NDJSON. `message_start` events carry input tokens; `message_delta` events carry cumulative output tokens. | Per API call |
| **Claude Code** (extended thinking) | No — degraded | Extended thinking disables `StreamEvent` emission. Tokens only at completion. | Post-completion only |
| **Codex CLI** | Yes | `--json` JSONL stream. `turn.completed` events carry per-turn usage deltas. | Per turn |
| **Gemini CLI** | No — partial | `--output-format stream-json` streams structured events, but token counts only appear in the final `result` event. | Post-completion only |

**Degraded mode:** When mid-run monitoring is unavailable (Gemini, Claude with extended thinking), rein checks context pressure post-completion only. The pressure zone is logged in the report for future task sizing decisions, but rein cannot intervene mid-run. For Gemini, OpenTelemetry export (`gen_ai.client.token.usage` metric) is a documented future path for mid-run token visibility.

### Zone Actions

**Yellow — Graceful Stop:**

1. Rein detects cumulative tokens crossing `green_max_pct` of context window
2. Wait for the agent's current turn to complete (do not kill mid-turn — partial tool calls leave inconsistent state)
3. Apply [Subprocess Termination Procedure](ARCHITECTURE.md#subprocess-termination-procedure) (graceful mode); `termination_reason=context_pressure`
4. Rein performs wrap-up (see [Rein Wrap-Up Protocol](#rein-wrap-up-protocol))
5. Optionally dispatch post-kill summary agent (see [Post-Kill Summary Agent](#post-kill-summary-agent))

**Red — Immediate Kill:**

1. Rein detects cumulative tokens crossing `yellow_max_pct` of context window
2. Apply [Subprocess Termination Procedure](ARCHITECTURE.md#subprocess-termination-procedure) (immediate mode); `termination_reason=context_pressure`
3. Rein performs wrap-up with whatever partial output was captured from the stream buffer
4. Post-kill summary agent not dispatched (context too degraded for useful summary)

**Post-completion (degraded mode):**

1. Agent completes normally (Gemini, Claude with extended thinking)
2. Rein parses final output, computes context pressure
3. Pressure zone logged in report — no mid-run intervention was possible
4. If red: rein logs a warning that the agent operated in degraded territory

**Timeout — Wall-Clock Limit Exceeded:**

1. Elapsed time since subprocess start exceeds `timeout_seconds` (default 300)
2. Apply [Subprocess Termination Procedure](ARCHITECTURE.md#subprocess-termination-procedure) (immediate mode); `termination_reason=timed_out`
3. Rein performs wrap-up (see [Rein Wrap-Up Protocol](#rein-wrap-up-protocol))
4. Post-kill summary agent not dispatched (equivalent to red-zone treatment)

Timeout and context pressure monitoring run concurrently — whichever fires first wins. If context pressure triggers a kill before the timeout, the timeout is cancelled. If the timeout fires first, context pressure monitoring stops.

### Rein Wrap-Up Protocol

When rein stops an agent (context pressure — yellow or red zone — or wall-clock timeout), it performs wrap-up itself — the agent is not trusted to wrap up because (a) there is no reliable "please wrap up" signal in CLI subprocess mode, and (b) at yellow/red pressure the agent's output quality is already degraded, and (c) on timeout the agent may be stuck or looping.

Wrap-up steps:

1. **Drain stream buffer** — capture all NDJSON/JSONL lines received up to the kill point
2. **Commit uncommitted changes** — `git add -A && git commit -m "rein: auto-commit at termination ({termination_reason})"` in the sandbox. The commit uses the agent's git identity (same `GIT_AUTHOR_*` / `GIT_COMMITTER_*` env vars set during sandbox setup — see [ARCHITECTURE.md — Agent Git Identity](ARCHITECTURE.md#agent-git-identity)), since the wrap-up is capturing the agent's uncommitted work.
3. **Write per-run log file** — includes: task ID, agent, model, zone at termination, token consumption metrics (cumulative input/output, peak utilization %, measurement method), number of turns, kill signal sent, captured stream summary
4. **Update PROGRESS.md** — append entry with: task ID, what was attempted (from task prompt), zone at termination, what was captured (git log summary, diff stat), whether a post-kill summary was generated
5. **Log termination metrics** — per-task record for trend analysis: `{ task_id, agent, model, context_window, estimated_tokens_used, utilization_pct, zone, measurement_method, turns, termination_reason, duration_seconds, timestamp }`. The `termination_reason` field uses the enum: `completed` (normal exit), `timed_out` (wall-clock limit), `context_pressure` (zone kill), or `error` (adapter failure)

### Post-Kill Summary Agent

After a yellow-zone stop (not red — at red the situation is too degraded for useful context), rein can optionally dispatch a fresh, short-lived agent to produce a semantic summary of what was accomplished.

This agent receives:
- The git log and diff stat from the stopped run
- The task's original prompt
- The captured stream output (last N messages)

Its job: write a concise summary of what was accomplished and what remains, suitable for inclusion in PROGRESS.md and for informing the next task dispatch.

The summary agent runs in a fresh context (no pressure), uses a small/fast model, and is budget-limited to a single turn. For Claude, the `--resume` trick is an alternative: resume the stopped session with `--max-turns 1` and "Summarize what you accomplished and what remains."

### Agent Prompt Engineering (Defense in Depth)

Context pressure monitoring is rein's responsibility. But as defense in depth, every task prompt should instruct the agent to:

- **Commit frequently** — after each meaningful change, not just at the end
- **Update PROGRESS.md after each commit** — record what was done and what remains
- **Log consistently** — write to the task log file after significant operations

This means even if rein kills the agent, most work is already persisted. The wrap-up only catches the last uncommitted delta. Task artifacts (JSON definitions) should be sized to fit within the green zone under normal conditions.

---

## Token Budget

Token budget monitoring — the 70k default, what counts toward the budget, budget status thresholds, and when to start fresh — is documented in [TOKENS.md](TOKENS.md).

The token budget is a **complementary** metric to context pressure. Context pressure tracks quality risk (tokens vs. context window). The token budget tracks resource consumption (tokens vs. spending limit). Both are monitored; context pressure drives intervention decisions, while the token budget informs cost management.

---

## Session Continuation (Per-Agent)

By default, each rein run is a fresh, isolated execution. But agents support session continuation for multi-turn workflows:

### Claude Code

```bash
# Get session ID from first run
session_id=$(claude -p "query" --output-format stream-json | jq -r 'select(.type=="result") | .session_id')

# Continue in that session
claude -p "next query" --resume "$session_id"

# Or continue the most recent session
claude -p "next query" --continue
```

**Note:** When resuming a session, context pressure carries forward from the previous execution. Rein must account for prior token consumption when computing pressure for the resumed session.

### Codex CLI

```bash
# Resume last session
codex exec resume --last "next task"

# Resume specific session
codex exec resume <SESSION_ID>
```

### Gemini CLI

No documented session resume mechanism. Each invocation is independent.

---

## Report Format

The structured JSON report format — schema, key fields, evaluation scoring, and output conventions — is documented in [REPORTS.md](REPORTS.md).

---

## Multi-Session Patterns

For tasks that need iteration:

1. **Break it down** — split a large task into sequential subtasks, each with its own budget. Size task artifacts to fit within the green zone.
2. **Fresh context per subtask** — each subtask gets a clean sandbox and a fresh agent session (zero context pressure).
3. **Carry forward artifacts** — use `files` in the next task's JSON to seed with output from the previous task.
4. **Carry forward learnings** — `.rein/LEARNINGS.md` in the project root accumulates operational knowledge across sessions. Each session reads learnings at start (injection) and writes discoveries back after its final verdict (extraction). Knowledge compounds — "tests need `--test-threads=1`" discovered in session 1 benefits all subsequent sessions. See [ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md).
5. **Monitor pressure trends** — if context pressure at completion is increasing across subtasks, the problem decomposition may need rethinking. The per-task metrics log enables this analysis.
