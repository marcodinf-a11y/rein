# ADR-005: NDJSON/JSONL Streaming Over Single JSON Output

## Status

Accepted

## Context

Each agent CLI can produce output in multiple formats. The two relevant options are:

- **Single JSON blob:** The agent writes one JSON object to stdout on completion. Simple to parse. But if the process is killed mid-execution (context pressure, timeout), the output is a truncated JSON string — unparseable and unrecoverable.
- **NDJSON/JSONL stream:** The agent writes one JSON object per line as events occur. Each line is independently parseable. If the process is killed, all complete lines are preserved; only the final incomplete line (if any) is lost.

Rein's core feature — real-time context pressure monitoring — requires reading token usage as the agent works, not after it finishes. This is only possible with streaming output.

## Decision

All three agents are configured in streaming mode:

| Agent | Flag | Format |
|-------|------|--------|
| Claude Code | `--output-format stream-json` | NDJSON |
| Codex CLI | `--json` | JSONL |
| Gemini CLI | `--output-format stream-json` | NDJSON |

Rein reads stdout line by line, parsing each line as an independent JSON object. Token usage events are extracted in real-time to feed the context pressure monitor. On subprocess kill, the stream buffer is drained — all complete lines are captured, the incomplete final line (if any) is discarded.

## Consequences

**Positive:**

- Enables real-time context pressure monitoring — the defining feature of rein.
- Line-recoverable on kill. Context pressure kills and timeouts preserve all work up to the last complete event. No "truncated JSON blob" failure mode.
- Parse failures are isolated to individual lines rather than corrupting the entire output.

**Negative:**

- More complex parsing than a single JSON blob. Each agent emits different event types with different schemas — the adapter must filter for relevant events.
- Gemini's stream mode only includes token counts in the final `result` event, not in intermediate events. Mid-run monitoring is unavailable for Gemini despite using the streaming format — the stream is still preferred for kill-recoverability.
- Claude's extended thinking mode disables stream event emission. The stream produces only the final result event, degrading to single-blob behavior.
