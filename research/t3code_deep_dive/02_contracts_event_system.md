# T3Code Deep Dive: Contracts & Event System

**March 2026**

This document analyzes T3Code's `contracts` package — its pure-types philosophy, Effect Schema foundation, the 47+ provider runtime event types, 20 orchestration event types, and the canonical item model. We compare this taxonomy with Rein's planned event architecture.

---

## 1. Contracts Package Philosophy

T3Code isolates all shared types into a dedicated `packages/contracts/` package that enforces a strict rule: **pure types, zero runtime logic.** The contracts package contains only Effect Schema definitions, TypeScript type exports, and enum declarations. No functions, no side effects, no I/O.

This design choice has several consequences:

- **Provider independence.** Any adapter (OpenAI, Anthropic, Google) imports the same canonical types. The adapter's job is to translate provider-specific wire formats into these shared schemas.
- **Tree-shakeable.** Because there is no runtime code, consumers pay only for the types they reference. Bundlers eliminate unused schemas entirely.
- **Single source of truth.** Every package in the monorepo — runtime, orchestrator, UI — depends on contracts. Schema changes propagate through the type system at compile time, not through runtime failures.
- **Versioned migration.** Legacy aliases coexist with current schemas, allowing gradual migration without breaking downstream consumers.

The philosophical commitment is clear: the contracts package is a **specification**, not an implementation.

---

## 2. Schema File Organization

The `packages/contracts/src/` directory contains 14+ schema files, each responsible for a distinct domain:

| File | Domain |
|------|--------|
| `provider-runtime-events.ts` | All provider-level events (47+ types) |
| `orchestration-events.ts` | Orchestration-level events (20 types) |
| `canonical-items.ts` | The 16-type item model |
| `provider-config.ts` | Provider configuration schemas |
| `thread.ts` | Thread and conversation schemas |
| `project.ts` | Project-level schemas |
| `session.ts` | Session lifecycle schemas |
| `auth.ts` | Authentication and credential schemas |
| `tool.ts` | Tool definition and invocation schemas |
| `model.ts` | Model metadata and capability schemas |
| `cost.ts` | Token usage and cost tracking schemas |
| `error.ts` | Structured error schemas |
| `common.ts` | Shared primitives (IDs, timestamps, enums) |
| `index.ts` | Barrel re-exports |

All schemas are defined using Effect Schema (formerly `@effect/schema`), which provides runtime validation, encoding/decoding, and TypeScript type inference from a single definition. This is a meaningful choice over alternatives like Zod or io-ts — Effect Schema integrates with the broader Effect ecosystem that T3Code uses for its runtime.

---

## 3. ProviderRuntimeEventBase

Every provider runtime event extends `ProviderRuntimeEventBase`, which carries a rich context envelope:

```
ProviderRuntimeEventBase
  eventId:       string    — UUID, unique per event instance
  provider:      string    — "openai" | "anthropic" | "google" | etc.
  threadId:      string    — conversation thread this event belongs to
  createdAt:     DateTime  — ISO 8601 timestamp
  turnId:        string    — groups events within a single agent turn
  itemId:        string    — links to the canonical item being produced
  requestId:     string    — correlates with the HTTP request to the provider
  providerRefs:  object    — provider-specific IDs (e.g., OpenAI's response ID)
  raw:           unknown   — the unprocessed provider payload, preserved for debugging
```

Key observations:

- **turnId** enables grouping all events from a single LLM invocation (prompt-in, tokens-streaming, response-complete) without relying on temporal ordering alone.
- **itemId** connects streaming events to the canonical item they are producing. A `content.delta` event and the eventual `content.done` event share the same itemId.
- **requestId** correlates events with the specific API call, which is critical when a single turn involves multiple requests (e.g., tool calls that trigger follow-up completions).
- **providerRefs** is deliberately untyped (`object`). Each adapter populates it with provider-specific identifiers. This avoids polluting the base schema with provider-specific fields while preserving traceability.
- **raw** preserves the original provider payload. This is a debugging escape hatch — when the canonical event model loses information during translation, the raw payload is always available for inspection.

This base schema is significantly richer than a minimal event envelope. The design reflects a lesson learned: **correlation is hard to add retroactively.** By embedding turnId, itemId, and requestId from the start, T3Code avoids the "which events belong together?" problem that plagues simpler event systems.

---

## 4. The 47+ ProviderRuntimeEvent Types

Provider runtime events are organized into eight categories. Each event type extends `ProviderRuntimeEventBase` with category-specific fields.

### 4.1 Session Events

Events governing the lifecycle of a provider session (connection establishment, configuration, teardown):

- `session.created`
- `session.configured`
- `session.terminated`
- `session.error`

### 4.2 Thread Events

Thread-level lifecycle within a session:

- `thread.created`
- `thread.resumed`
- `thread.archived`

### 4.3 Turn Events

A "turn" is a single prompt-response cycle. Turn events bracket the LLM invocation:

- `turn.started`
- `turn.completed`
- `turn.failed`
- `turn.cancelled`

### 4.4 Item Events

Item-level lifecycle — creation, update, completion of canonical items:

- `item.created`
- `item.updated`
- `item.completed`
- `item.deleted`
- `item.failed`

### 4.5 Content Events

Fine-grained streaming events for content production:

- `content.delta` — incremental text/token chunk
- `content.done` — content block finalized
- `content.truncated` — content cut short (context limit)
- `content.annotation` — metadata attached to content (citations, etc.)

### 4.6 Request Events

HTTP-level observability for provider API calls:

- `request.started`
- `request.completed`
- `request.failed`
- `request.retried`
- `request.rate_limited`
- `request.timeout`

### 4.7 Auth Events

Authentication lifecycle:

- `auth.token_refreshed`
- `auth.token_expired`
- `auth.failed`

### 4.8 Runtime/Diagnostic Events

Operational events for monitoring and debugging:

- `runtime.started`
- `runtime.stopped`
- `runtime.error`
- `runtime.warning`
- `diagnostic.token_usage` — per-request token counts
- `diagnostic.latency` — timing measurements
- `diagnostic.cost` — cost calculation events
- `diagnostic.context_window` — context utilization snapshots

Plus additional specialized variants within each category (e.g., tool-specific item events, multi-modal content deltas), bringing the total to 47+.

**Analysis.** The granularity here is notable. Most agentic frameworks define 5-10 event types and overload a generic "status" or "log" event for everything else. T3Code's approach means that consumers can subscribe to precisely the events they care about — a cost dashboard subscribes to `diagnostic.cost`, a streaming UI subscribes to `content.delta`, a retry policy subscribes to `request.rate_limited`. The tradeoff is schema surface area: 47+ event types require discipline to maintain and version.

---

## 5. CanonicalItemType Enum

The `CanonicalItemType` enum defines the 16 types that any provider response can produce:

```
message        — text response from the model
tool_call      — request to invoke a tool
tool_result    — output from a tool invocation
file           — file content (read or generated)
image          — image content (generated or referenced)
error          — error from the model or provider
status         — status update (thinking, working, etc.)
plan           — a structured plan (multi-step)
plan_step      — individual step within a plan
checkpoint     — snapshot marker for rollback
rollback       — revert to a previous checkpoint
annotation     — metadata attached to another item
citation       — source reference
code_block     — executable code (distinct from message text)
diff           — file modification (patch format)
command        — shell command execution
```

Key design decisions:

- **plan and plan_step as first-class types.** This signals that T3Code treats planning as a native capability, not an emergent behavior. The orchestrator can reason about plans structurally rather than parsing them from free-text responses.
- **checkpoint and rollback.** These support T3Code's recovery model. A checkpoint item marks a known-good state; a rollback item signals reversion. This is the "Beads pattern" identified in the Gas Town deep dive, implemented at the type level.
- **diff as a distinct type.** Rather than embedding diffs in `message` or `file` items, T3Code gives them a dedicated type. This allows the UI to render diffs with syntax highlighting and the orchestrator to apply them programmatically.
- **command as a type.** Shell command execution is modeled as an item, giving it the same lifecycle (created, completed, failed) as any other output.

The enum is intentionally extensible — new types can be added without breaking existing consumers, since unknown types are handled by a fallback path in every consumer.

---

## 6. OrchestrationEvent Types

Orchestration events operate at a higher abstraction layer than provider events. They describe what the orchestrator is doing, not what the provider is returning.

### 6.1 EventBase

Every orchestration event extends `EventBase`:

```
EventBase
  sequence:          number  — monotonically increasing event counter
  causationEventId:  string  — the event that directly caused this one
  correlationId:     string  — groups all events from the same user action
```

This is a textbook event-sourcing envelope:

- **sequence** provides total ordering within a stream. Unlike timestamps, sequence numbers are gap-free and monotonic, making them reliable for replay and consistency checks.
- **causationEventId** builds an explicit causal chain. When a `thread.command.dispatched` event triggers a `thread.turn.started` event, the turn event's causationEventId points to the dispatch event. This enables full causal graph reconstruction.
- **correlationId** groups all events that trace back to a single user action. A user clicking "run task" might generate dozens of events across project, thread, and provider layers — they all share the same correlationId.

### 6.2 The 20 OrchestrationEvent Types

Organized by domain:

**Project CRUD:**
- `project.created`
- `project.updated`
- `project.deleted`
- `project.archived`

**Thread CRUD:**
- `thread.created`
- `thread.updated`
- `thread.deleted`
- `thread.archived`

**Thread Commands (with causation tracking):**
- `thread.command.dispatched`
- `thread.command.completed`
- `thread.command.failed`
- `thread.command.cancelled`

**Orchestration Lifecycle:**
- `orchestration.started`
- `orchestration.paused`
- `orchestration.resumed`
- `orchestration.completed`
- `orchestration.failed`

**Configuration:**
- `config.updated`
- `config.validated`
- `config.error`

The command events are particularly interesting. Each `thread.command.dispatched` event carries the full command payload — what to do, which thread, with what parameters. The `completed`/`failed`/`cancelled` events reference back via `causationEventId`. This means the entire command lifecycle is reconstructable from the event log without any external state.

---

## 7. Legacy Compatibility Aliases

T3Code maintains a set of type aliases that map V1 event names to their V2 equivalents:

```typescript
// V1 name → V2 schema
export const ProviderEvent = ProviderRuntimeEvent;
export const OrchestratorEvent = OrchestrationEvent;
export const ItemType = CanonicalItemType;
```

These aliases live in a dedicated `legacy.ts` file (or a `compat` section of the barrel export). They are annotated with `@deprecated` JSDoc tags so that IDE tooling flags usage. The migration strategy is clear:

1. V2 schemas are the canonical definitions.
2. V1 aliases re-export V2 schemas under old names.
3. Consumers migrate at their own pace.
4. Aliases are removed in a future major version.

This is a pragmatic approach to schema evolution. The alternative — versioned schema namespaces like `v1.ProviderEvent` and `v2.ProviderRuntimeEvent` — adds import complexity that T3Code apparently decided was not worth the cost.

---

## 8. Comparison with Rein's Planned Event Taxonomy

Rein's current architecture does not define a formal event system. Context pressure monitoring and quality evaluation produce structured data, but this data is not modeled as a typed event stream. Comparing T3Code's approach highlights several considerations for Rein.

### 8.1 Where T3Code's Model Exceeds Rein's Needs

- **Provider-level streaming events.** Rein delegates to Claude Code as an opaque subprocess. Rein does not intercept streaming tokens or manage HTTP connections to providers. The `content.delta`, `request.*`, and `auth.*` event categories are irrelevant to Rein's architecture.
- **Multi-provider normalization.** T3Code's contracts exist partly to normalize across OpenAI, Anthropic, and Google APIs. Rein's MVP targets Claude Code only. The normalization layer adds complexity without benefit at this stage.
- **UI-driven event consumption.** T3Code has a rich UI that consumes events for real-time display. Rein is CLI-first; its event consumers are the monitoring loop and log output.

### 8.2 Where Rein Should Learn from T3Code

- **Causation and correlation IDs.** Rein's planned stagnation detection and quality evaluation would benefit from explicit causal chains. When a kill decision is made, being able to trace back through causationEventId to the specific context pressure reading that triggered it would improve debuggability.
- **Sequence numbers over timestamps.** Rein currently relies on temporal ordering. Monotonic sequence numbers are more reliable, especially when events are generated across worktree subprocesses that may have clock skew.
- **Canonical item types.** Rein does not currently model what the agent produces — it monitors token counts and quality signals. Adopting a minimal item type enum (message, tool_call, tool_result, error, status) would enable richer quality evaluation. For example, a turn that produces only `status` items and no `tool_call` or `message` items is a strong stagnation signal.
- **Checkpoint/rollback as types.** Rein's worktree isolation provides implicit rollback (discard the worktree). But modeling checkpoints explicitly would support the crash-recoverable task state identified in the Gas Town deep dive.

### 8.3 Recommended Adoption for Rein

**Adopt now (MVP):**
- Define a minimal `EventBase` with sequence number and correlation ID. Even if the initial event set is small (task.started, task.completed, task.failed, context.pressure_reading, quality.evaluation), the envelope should be right from the start.
- Add causation tracking to kill decisions: the kill event should reference the pressure reading that triggered it.

**Adopt later (post-MVP):**
- Canonical item type enum, starting with the 5-6 types relevant to Claude Code output.
- Full event sourcing with replay capability.
- Provider-level events if/when Rein adds direct provider adapters.

**Do not adopt:**
- 47+ event types. Rein's architecture does not require this granularity. Start with fewer than 10 and expand based on actual consumer needs.
- Effect Schema as the schema system. Rein is Python-based; Pydantic or dataclasses serve the same purpose without introducing a TypeScript dependency pattern.

---

## 9. Architectural Takeaways

T3Code's contracts package demonstrates a mature approach to event-driven architecture in an agentic system. The key lessons are:

1. **Invest in the envelope early.** The `ProviderRuntimeEventBase` and `EventBase` schemas carry correlation, causation, and ordering metadata that would be expensive to retrofit. Rein should define its event envelope before writing event producers.

2. **Separate provider events from orchestration events.** These serve different consumers at different abstraction levels. Mixing them creates events that are too detailed for the orchestrator and too abstract for the provider adapter.

3. **Pure types, zero logic.** The contracts package's discipline — no runtime code, only schemas — prevents circular dependencies and keeps the type system as the single source of truth. In Python, this maps to a `rein.contracts` or `rein.types` module containing only dataclasses and enums.

4. **Plan for migration.** Legacy aliases are cheap insurance. When Rein's event schemas evolve (and they will), maintaining backward-compatible aliases avoids breaking consumers during transition periods.

5. **Granularity is a spectrum.** T3Code chose high granularity (47+ provider events, 20 orchestration events) because it has many consumers with diverse needs. Rein should choose the granularity that matches its consumer count — currently low — and expand incrementally.

---

## 10. Source References

- `packages/contracts/src/provider-runtime-events.ts` — ProviderRuntimeEventBase and all provider event schemas
- `packages/contracts/src/orchestration-events.ts` — EventBase and orchestration event schemas
- `packages/contracts/src/canonical-items.ts` — CanonicalItemType enum and item schemas
- `packages/contracts/src/legacy.ts` — V1 compatibility aliases
- `packages/contracts/src/index.ts` — barrel exports
- `packages/contracts/package.json` — zero runtime dependencies confirmed
