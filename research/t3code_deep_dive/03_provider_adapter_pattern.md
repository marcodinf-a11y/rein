# T3Code Deep Dive: Provider Adapter Pattern

**March 2026**

This document analyzes T3Code's provider adapter abstraction — the `ProviderAdapterShape<E>` interface, its registry, the event streaming pipeline, and the concrete adapter implementations for Codex and Claude Code. We compare this with Rein's planned `AgentAdapter` protocol and extract actionable lessons.

---

## 1. The ProviderAdapterShape Interface

T3Code defines a single generic interface that every provider must implement:

```typescript
interface ProviderAdapterShape<E extends ProviderKind> {
  // Lifecycle
  start(input: ProviderSessionStartInput): Promise<ProviderSessionHandle>
  stop(sessionId: string): Promise<void>
  interrupt(sessionId: string): Promise<void>
  resume(sessionId: string): Promise<void>
  rollback(sessionId: string, checkpointId: string): Promise<void>

  // Communication
  sendMessage(sessionId: string, message: UserMessage): Promise<void>

  // Metadata
  getModels(): Promise<ProviderModelInfo[]>
  getHealth(): Promise<ProviderHealth>
  getSessionState(sessionId: string): Promise<ProviderSessionState>
  getCapabilities(): ProviderCapabilities
  getDefaultModel(): string

  // Observation
  streamEvents(sessionId: string): Stream<ProviderRuntimeEvent>
}
```

Eleven methods plus the event stream. This is a wide interface by deliberate design — T3Code wants each adapter to declare its full operational surface up front, rather than discovering capabilities at runtime.

---

## 2. Why 11 Methods

The method count reflects T3Code's ambition to support both stateless CLI wrappers and stateful SDK integrations from the same interface. Breaking them down:

### 2.1 Lifecycle Group (start, stop, interrupt, resume, rollback)

Five methods dedicated to session lifecycle. This is more than most orchestrators need because T3Code supports mid-session intervention:

- **start** spawns the subprocess or SDK client, returns a handle with session ID
- **stop** is clean termination — flushes buffers, waits for graceful exit
- **interrupt** pauses the agent mid-turn without terminating
- **resume** continues after an interrupt
- **rollback** restores to a named checkpoint

The interrupt/resume/rollback trio is unusual. Most orchestrators treat agents as fire-and-forget. T3Code wants interactive control, which makes sense for its web UI where a user might pause an agent, review partial work, then continue.

### 2.2 Communication (sendMessage)

A single method for injecting user messages into a running session. This enables multi-turn conversation through the adapter rather than requiring a restart. The message is typed — it carries structured content blocks, not raw strings.

### 2.3 Metadata Group (getModels, getHealth, getSessionState, getCapabilities, getDefaultModel)

Five query methods. These serve the frontend: model selectors, health indicators, capability-driven UI toggles (e.g., hide the rollback button if the adapter reports `rollback: false` in capabilities).

---

## 3. The streamEvents Channel

The core observation mechanism. Every adapter must expose a `Stream<ProviderRuntimeEvent>` that the orchestrator subscribes to. This is the only way information flows out of the adapter during execution.

### 3.1 Event Categories

T3Code's `ProviderRuntimeEventV2` defines 45 event types across 9 categories:

| Category | Example Events | Purpose |
|----------|---------------|---------|
| session | session.started, session.error | Lifecycle boundaries |
| thread | thread.created | Conversation threading |
| turn | turn.started, turn.completed | Agent turn boundaries |
| item | item.text.delta, item.text.done | Streaming content |
| request | request.started, request.completed | API call tracking |
| task | task.started, task.completed | Structured task progress |
| tool | tool.started, tool.result | Tool use observation |
| auth | auth.required | Permission prompts |
| diagnostic | diagnostic.log, diagnostic.metric | Internal telemetry |

Every event carries causation metadata: `commandId`, `causationEventId`, `correlationId`, and a monotonic `sequence` number. This enables full causal reconstruction of any session.

### 3.2 The Event Pipeline

The pipeline from raw process output to consumer stream has four stages:

```
Process stdout
    |
    v
JSON-RPC parser (line-buffered)
    |
    v
mapToRuntimeEvents() — adapter-specific mapping
    |
    v
AsyncQueue<ProviderRuntimeEvent>
    |
    v
Stream (AsyncIterable) — exposed via streamEvents()
    |
    v
PubSub — fans out to multiple subscribers
```

Each stage is independently testable. The queue acts as a backpressure buffer — if consumers are slow, events accumulate in the queue rather than blocking the subprocess. The PubSub layer allows multiple subscribers (UI, logger, monitor) to observe the same stream without interference.

### 3.3 Error Events vs Exceptions

Adapter methods do not throw exceptions for agent-level errors. Instead, errors surface as `session.error` events in the stream. Only infrastructure failures (process spawn failure, SDK initialization error) throw. This separation is important: it means the consumer loop never needs try/catch around event processing, and error events carry the same causation metadata as everything else.

---

## 4. ProviderAdapterRegistry

The registry is a simple map from `ProviderKind` enum values to adapter factory functions:

```typescript
type ProviderKind = "codex" | "claude"

class ProviderAdapterRegistry {
  private adapters: Map<ProviderKind, () => ProviderAdapterShape<ProviderKind>>

  register(kind: ProviderKind, factory: () => ProviderAdapterShape<ProviderKind>): void
  get(kind: ProviderKind): ProviderAdapterShape<ProviderKind>
  listAvailable(): ProviderKind[]
}
```

The registry uses factory functions rather than singleton instances, so each session gets a fresh adapter. This avoids shared state between sessions — a lesson learned from early bugs where one session's interrupt signal leaked to another.

Currently only two providers are registered. The registry pattern exists to make adding a third (Gemini, Aider, etc.) a single-file change with no modifications to the orchestrator core.

---

## 5. CodexAdapter Implementation

The Codex adapter communicates with the OpenAI Codex CLI via JSON-RPC over subprocess stdio.

### 5.1 Subprocess Communication

Codex CLI exposes a `--json-rpc` mode where it reads JSON-RPC requests from stdin and writes events to stdout. The adapter:

1. Spawns `codex --json-rpc` with environment variables for API keys
2. Writes `start`, `sendMessage`, `interrupt` as JSON-RPC method calls to stdin
3. Reads stdout line-by-line, parsing each line as a JSON-RPC notification
4. Maps notifications to `ProviderRuntimeEvent` via a `codexEventMap` lookup table

### 5.2 Event Mapping

Codex emits its own event vocabulary (e.g., `codex.response.chunk`, `codex.tool_call.start`). The adapter maintains a mapping table:

```typescript
const codexEventMap = {
  "codex.response.chunk": "item.text.delta",
  "codex.response.done": "item.text.done",
  "codex.tool_call.start": "tool.started",
  "codex.tool_call.result": "tool.result",
  // ... 20+ mappings
}
```

Events that have no mapping are emitted as `diagnostic.log` with the raw payload, ensuring nothing is silently dropped.

### 5.3 Error Normalization

Codex errors arrive in various formats — JSON-RPC error responses, stderr output, non-zero exit codes. The adapter normalizes all of these into `session.error` events with a consistent schema:

```typescript
{
  type: "session.error",
  error: {
    code: "PROVIDER_ERROR" | "PROCESS_CRASH" | "TIMEOUT",
    message: string,
    raw: unknown  // original error payload
  }
}
```

This normalization is one of the adapter's most valuable functions. Consumers never need to understand Codex-specific error formats.

---

## 6. ClaudeCodeAdapter (PR #179)

The Claude Code adapter, merged in PR #179, takes a fundamentally different approach from CodexAdapter. Instead of subprocess JSON-RPC, it uses the `@anthropic-ai/claude-agent-sdk` directly.

### 6.1 SDK Integration

The SDK provides a typed client that handles:

- Session creation and management
- Message streaming with structured events
- Built-in interrupt/resume support
- Checkpoint management for rollback

This means the ClaudeCodeAdapter is thinner than CodexAdapter — it delegates lifecycle management to the SDK rather than implementing it via process signals.

### 6.2 Interrupt/Resume/Rollback

The Claude Agent SDK natively supports these operations, which made ClaudeCodeAdapter the first adapter to fully implement all 11 interface methods. CodexAdapter stubs `rollback` with a `NotSupported` error.

- **interrupt**: Calls `sdk.interrupt()`, which signals the agent to stop at the next safe point
- **resume**: Calls `sdk.resume()` with the session handle, picking up from the interruption point
- **rollback**: Calls `sdk.rollback(checkpointId)`, restoring conversation state to a named snapshot

### 6.3 Event Mapping

The SDK emits events in a format closer to T3Code's internal vocabulary, so the mapping layer is simpler — mostly structural transformation (flattening nested objects, adding causation IDs) rather than semantic translation.

---

## 7. ProviderSessionDirectory

T3Code persists session metadata to disk via `ProviderSessionDirectory`. Each session gets a directory containing:

```
sessions/
  {session-id}/
    meta.json          # start time, provider, model, status
    events.jsonl       # full event stream (append-only)
    checkpoints/       # rollback snapshots (adapter-specific)
```

The directory serves two purposes:

1. **Crash recovery** — on restart, T3Code scans the sessions directory, finds sessions with status `running`, and attempts to reconnect or clean up
2. **Replay** — the `events.jsonl` file is a complete record that can be replayed through the UI for debugging

The adapter writes to this directory through a `SessionWriter` injected at construction time. The adapter itself never reads from it — reading is the orchestrator's responsibility.

---

## 8. ProviderHealth

Each adapter implements `getHealth()` returning a structured health report:

```typescript
interface ProviderHealth {
  status: "healthy" | "degraded" | "unavailable"
  latency_ms: number | null
  last_check: ISO8601
  details: {
    binary_found: boolean
    binary_version: string | null
    api_reachable: boolean
    auth_valid: boolean
  }
}
```

Health checks run on adapter registration and periodically during idle time. The frontend uses this to show provider status indicators and disable unavailable providers.

For CodexAdapter, health checks involve:
- Verifying `codex` binary exists on PATH
- Running `codex --version` to get the version string
- Making a lightweight API call to verify authentication

For ClaudeCodeAdapter, health checks are simpler since the SDK handles connection management internally.

---

## 9. Comparison with Rein's AgentAdapter Protocol

Rein's planned `AgentAdapter` in `src/rein/agents/base.py` is deliberately narrower than T3Code's interface:

| Capability | T3Code | Rein (Planned) |
|-----------|--------|----------------|
| Start/Stop | Yes | Yes |
| Interrupt | Yes | Yes (SIGTERM cascade) |
| Resume | Yes | No (single-shot sessions) |
| Rollback | Yes | No |
| Send message | Yes | No (prompt at start only) |
| Stream events | 45 event types | Stdout line stream |
| Health check | Structured | Not planned for R1 |
| Model registry | Per-adapter | Global config |
| Session persistence | Directory-based | Not planned for R1 |

Rein's narrower surface is appropriate for its scope. Key differences:

1. **No mid-session interaction.** Rein dispatches a task and monitors externally. T3Code enables ongoing conversation through the adapter. Rein's model is simpler and sufficient for autonomous coding tasks.

2. **Process-level rather than SDK-level control.** Rein's adapters wrap CLI subprocesses and observe stdout. T3Code's ClaudeCodeAdapter uses the SDK directly. Rein's approach is more uniform across providers but loses access to structured events that SDKs provide.

3. **No rollback.** Rein uses workspace isolation (worktrees, tempdirs) as its "rollback" — if a task fails, the workspace is disposable. T3Code needs rollback because sessions are long-lived and interactive.

---

## 10. Lessons for Rein

### 10.1 Adopt: Event Normalization Layer

T3Code's `mapToRuntimeEvents()` step is the highest-value pattern here. Even with Rein's simpler stdout-based observation, different agents emit different formats. A normalization step between "raw agent output" and "Rein internal events" prevents agent-specific parsing from leaking into the monitor.

Rein already plans per-adapter parsers. The lesson is to ensure they all emit the same event types — a shared `AgentEvent` union — rather than returning adapter-specific structures.

### 10.2 Adopt: Error Normalization

T3Code's three-category error model (`PROVIDER_ERROR`, `PROCESS_CRASH`, `TIMEOUT`) with the raw payload preserved is clean and sufficient. Rein should normalize agent failures the same way rather than passing raw exit codes and stderr up to the orchestrator.

### 10.3 Adopt: Factory-Based Registry

The factory function pattern (rather than singleton adapters) prevents shared state bugs. If Rein ever runs tasks concurrently, this becomes essential. Even in sequential mode, it simplifies testing — each test gets a fresh adapter instance.

### 10.4 Consider: Capabilities Declaration

T3Code's `getCapabilities()` lets the orchestrator query what an adapter supports. Rein's protocol could include a class-level `capabilities` dict or set indicating which optional features the adapter supports (e.g., `{"stream_tokens", "structured_events"}`). This is low-effort and prevents runtime errors from calling unsupported methods.

### 10.5 Skip: Interactive Methods (sendMessage, resume, rollback)

These serve T3Code's web UI use case. Rein's single-shot dispatch model does not need them. Adding them would complicate the protocol without benefit.

### 10.6 Skip: Session Persistence Directory

Rein's workspaces already serve as the persistence boundary. A separate session directory would be redundant. The JSONL event log (planned for R3) can live in the workspace output directory.

---

## 11. Summary

T3Code's provider adapter pattern is well-engineered for its use case: a web GUI wrapping multiple interactive agent backends. The 11-method interface is wide but justified by the interactive session model. The event streaming pipeline — with its four-stage architecture, backpressure queue, and PubSub fan-out — is the most transferable pattern.

For Rein, the key takeaways are:

| Pattern | Action | Priority |
|---------|--------|----------|
| Event normalization layer | Adopt — shared `AgentEvent` union across adapters | P1 (R1) |
| Error normalization | Adopt — three-category model with raw payload | P1 (R1) |
| Factory-based adapter registry | Adopt — avoid shared state | P1 (R1) |
| Capabilities declaration | Consider — low-effort, prevents runtime errors | P2 (R1) |
| Interactive methods | Skip — not needed for single-shot dispatch | -- |
| Session persistence directory | Skip — workspaces serve this role | -- |
| Health check protocol | Defer to R2 — useful but not blocking | P2 (R2) |

The biggest gap between T3Code and Rein is not the interface width but the observation depth. T3Code's 45 structured event types give it fine-grained visibility into agent behavior. Rein's stdout-based monitoring is coarser but more portable. As agent SDKs mature and expose richer event streams, Rein's adapter protocol should be ready to consume them — which means the normalization layer must be designed for extension from the start.
