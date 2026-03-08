# ADR-006: Subprocess Termination Procedure

## Status

Accepted

## Context

When rein decides to stop an agent (context pressure zone action or wall-clock timeout), it must terminate a running subprocess. The termination must balance three concerns:

- **Data preservation:** Capture as much output as possible before the process dies.
- **State consistency:** Avoid killing mid-tool-call, which can leave files half-written or git state corrupted.
- **Timeliness:** A stuck or runaway agent must actually stop. Sending a polite signal and waiting forever defeats the purpose.

Each agent CLI responds differently to Unix signals. Codex treats SIGINT as its canonical stop signal (emitting a `TurnAborted` event), while Claude and Gemini respond to SIGTERM.

## Decision

Two termination modes sharing a common signal sequence:

**Graceful mode** (yellow zone context pressure):
1. Wait for the agent's current turn to complete (do not kill mid-turn).
2. Then apply the signal sequence below.

**Immediate mode** (red zone context pressure, timeout):
1. Apply the signal sequence immediately — do not wait for the current turn.

**Signal sequence:**
1. Send `SIGINT` for Codex (triggers `TurnAborted` event), `SIGTERM` for Claude and Gemini.
2. Wait up to 5 seconds for the process to exit.
3. If still alive after 5 seconds, send `SIGKILL`.

**Post-kill:** Drain the stdout buffer. All complete NDJSON/JSONL lines are captured and parseable. Discard any incomplete final line. Record the process `returncode` as-is (typically `-15` for SIGTERM, `-9` for SIGKILL, `130` for Codex SIGINT).

## Consequences

**Positive:**

- Graceful mode prevents half-completed tool calls from corrupting sandbox state. The agent finishes its current action before stopping.
- Per-agent signal selection respects each CLI's documented behavior. Codex gets a clean `TurnAborted` event on SIGINT.
- The 5-second grace period allows agents to flush output buffers. SIGKILL is the backstop for truly stuck processes.
- Streaming output (ADR-005) ensures all complete lines are preserved regardless of termination mode.

**Negative:**

- Graceful mode has unbounded wait time — a very long turn (e.g., large file write) delays termination. No timeout on the "wait for turn" step is currently defined.
- The 5-second grace period is a fixed constant. Some agents may need more time to flush; others could be killed faster. This is not configurable.
- SIGKILL cannot be caught — the agent gets no chance to clean up. This is intentional (it's the last resort) but means post-SIGKILL state may be inconsistent.
