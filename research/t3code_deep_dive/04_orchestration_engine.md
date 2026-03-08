# T3Code Deep Dive: Orchestration Engine

**March 2026**

This document analyzes T3Code's orchestration engine, which implements a full event-sourced CQRS architecture for managing coding agent sessions. The engine is the central nervous system of T3Code: it accepts commands, produces events, persists state, and drives projections that feed the UI and provider adapters.

---

## 1. Architectural Overview

T3Code's orchestration layer follows the event sourcing + CQRS (Command Query Responsibility Segregation) pattern. The data flow is:

```
Commands --> Decider --> Events --> Store --> Projectors --> Read Models
              |                      |
              |                      +--> Reactors (side effects)
              |
              +-- validates, applies business rules
```

The command side and query side are fully separated. Commands mutate state by producing events. Queries read from materialized projections that are updated asynchronously as events arrive. This separation means the write path never blocks on read-model construction, and read models can be rebuilt from the event log at any time.

---

## 2. Command Side

### 2.1 OrchestrationEngine

The `OrchestrationEngine` is the command entry point. Its responsibilities:

1. **Command dispatch** -- accepts typed command objects, validates them, and routes each command to the appropriate decider
2. **Precondition checks** -- rejects commands that violate invariants before they reach the decider (e.g., starting a session that is already running)
3. **Event persistence** -- takes the events emitted by the decider and appends them to the event store in a single transaction
4. **Reactor notification** -- after persistence, notifies registered reactors so side effects can execute

The engine itself is stateless between command invocations. All state lives in the event store and in projections derived from it. This means the engine can be restarted without losing anything -- it rebuilds its understanding of the world by replaying events.

### 2.2 Decider

The decider is a pure function at the conceptual level: given the current state (derived from past events) and a command, it produces zero or more new events. The decider encodes all domain rules:

- Can this session transition from its current state to the requested state?
- Does the referenced thread/project exist?
- Are the provider parameters valid?

Because the decider is pure (no I/O, no side effects), it is straightforward to test. You feed it a sequence of historical events to establish state, then a command, and assert on the events it produces.

### 2.3 Command Validation

Commands are validated at two levels:

1. **Structural validation** -- the command object itself enforces type constraints (required fields, value ranges, enum membership)
2. **Domain validation** -- the decider checks the command against current aggregate state (e.g., you cannot stop a session that is not running)

Invalid commands produce typed errors rather than events. The engine catches these and surfaces them to the caller without persisting anything.

---

## 3. Query Side

### 3.1 ProjectionPipeline

The `ProjectionPipeline` subscribes to the event store and feeds events to registered projectors. It handles:

- **Ordering guarantees** -- events are delivered to projectors in store order
- **Idempotency** -- each projector tracks its last-processed event position, so replaying the pipeline from scratch produces the same read models
- **Error isolation** -- a failing projector does not block other projectors from processing events

The pipeline can run in two modes: catch-up (replaying historical events to rebuild projections) and live (tailing new events as they are appended).

### 3.2 Projectors and Read Models

Projectors transform events into denormalized read models optimized for specific query patterns. The key projections are:

| Projector | Read Model | Purpose |
|-----------|-----------|---------|
| ThreadProjector | Thread list/detail | Current state of each thread, message counts, last activity |
| ProjectProjector | Project list/detail | Project metadata, associated threads, configuration |
| SessionProjector | Session state | Current session status, timing, provider info |

Read models are stored in SQLite alongside the event store (but in separate tables). They are disposable -- if a projection table is corrupted or its schema changes, it can be dropped and rebuilt by replaying all events through the projector.

---

## 4. Reactors

Reactors respond to events by triggering side effects. Unlike projectors (which build read models), reactors perform actions that affect the outside world.

### 4.1 ProviderCommandReactor

The `ProviderCommandReactor` translates domain events into provider adapter commands. When the orchestration engine emits a `SessionStarted` event, this reactor:

1. Resolves the provider adapter (e.g., Claude Code) from the session configuration
2. Constructs the appropriate provider command (spawn process, inject prompt, configure workspace)
3. Dispatches the command to the adapter
4. Listens for adapter responses and feeds them back as new commands to the orchestration engine

This reactor is the bridge between the domain model (which knows about sessions, threads, and projects) and the provider layer (which knows about processes, tokens, and API calls). The domain model never references provider-specific concepts directly.

### 4.2 CheckpointReactor

The `CheckpointReactor` handles checkpoint creation and restoration. It reacts to:

- **Session interrupted/stopped events** -- creates a checkpoint capturing the current workspace state, git position, and session metadata
- **Session restore commands** -- reads a checkpoint and reconstructs the workspace to the captured state before allowing the session to resume

Checkpoints enable Rein-relevant patterns like context rotation: a session can be killed at a context pressure threshold, checkpointed, and restarted from the checkpoint with a fresh context window.

---

## 5. Persistence Layer

### 5.1 SQLite and Migrations

T3Code uses SQLite for all persistence. The schema is managed through 13 migrations that build up:

- **Event store table** -- the append-only log of all events, with columns for sequence number, aggregate ID, event type, payload (JSON), and timestamp
- **Projection tables** -- materialized read models for threads, projects, and sessions
- **Checkpoint tables** -- workspace snapshots tied to session events

SQLite is a pragmatic choice for a CLI tool: zero deployment overhead, single-file database, ACID transactions, and sufficient performance for the event volumes a coding agent produces. The entire state of a T3Code installation lives in one `.db` file.

### 5.2 Event Store Design

The event store is an append-only table. Events are never updated or deleted. Each event has:

- A globally unique sequence number (monotonically increasing)
- An aggregate ID (identifies which entity the event belongs to)
- An event type discriminator
- A JSON payload containing event-specific data
- A timestamp

This design means the event store is a complete audit log. You can answer questions like "what happened to session X" by filtering events by aggregate ID and reading them in sequence order.

---

## 6. Error Model

T3Code defines six typed error classes for the orchestration layer:

| Error Class | Trigger |
|-------------|---------|
| `ThreadNotFound` | Command references a thread ID that does not exist in the event store |
| `ProjectNotFound` | Command references a project ID that does not exist |
| `InvalidCommand` | Command fails structural or domain validation |
| `ProviderError` | The provider adapter reports a failure (process crash, API error, timeout) |
| `CheckpointError` | Checkpoint creation or restoration fails (I/O error, corrupt snapshot) |
| `StoreError` | Event store persistence fails (SQLite write error, constraint violation) |

These errors are structured, not string-based. Each carries a machine-readable error code and contextual data (e.g., which thread ID was not found). This allows callers to pattern-match on errors and take appropriate recovery action rather than parsing error messages.

The error hierarchy is flat -- there is no inheritance chain. Each error class is independent. This avoids the common problem of exception hierarchies where catching a parent class accidentally swallows unrelated errors.

---

## 7. Session State Machine

Sessions follow a well-defined state machine:

```
                +----------+
                |   idle   |
                +----+-----+
                     |
                  [start]
                     |
                +----v-----+
                | starting  |
                +----+-----+
                     |
              [provider ready]
                     |
                +----v-----+
         +----->| running   |<------+
         |      +----+-----+       |
         |           |              |
         |     [interrupt]     [resume]
         |           |              |
         |      +----v---------+   |
         |      | interrupted  +---+
         |      +----+---------+
         |           |
         |        [stop]
         |           |
         |      +----v-----+
         |      | stopped   |
         |      +-----------+
         |
         |    [unrecoverable error]
         |           |
         |      +----v-----+
         +------+  error   |
        [retry]  +-----------+
                     |
                  [reset]
                     |
                +----v-----+
                |  ready    |
                +-----------+
```

State transitions:

- **idle -> starting**: A start command is accepted. The engine begins provider initialization.
- **starting -> running**: The provider adapter signals readiness. The session is now executing.
- **running -> interrupted**: A context pressure threshold is hit, or the user manually interrupts. The session pauses but retains its checkpoint.
- **interrupted -> running**: The session resumes from its checkpoint (possibly with a rotated context window).
- **interrupted -> stopped**: The user or system decides not to resume. Terminal state.
- **running -> error**: An unrecoverable provider failure occurs.
- **error -> running**: A retry is attempted (only for transient errors).
- **error -> ready**: The error is acknowledged and the session is reset for a fresh start.

The state machine is enforced by the decider. A command that would cause an invalid transition is rejected before any event is produced. This means the event store contains only valid state transitions -- you can never arrive at an inconsistent session state by replaying events.

---

## 8. Relevance to Rein

T3Code's orchestration engine is significantly more complex than what Rein needs for its MVP. The event sourcing pattern provides capabilities (full audit log, projection rebuilds, temporal queries) that are valuable for a long-running IDE-integrated tool but excessive for a CLI harness focused on context rotation.

**Patterns worth studying:**

- **Session state machine** -- Rein's session lifecycle (dispatch, monitor, kill, rotate) maps naturally to a state machine. Defining valid transitions explicitly prevents bugs where a session is killed twice or monitored after completion.
- **Reactor pattern for side effects** -- separating "what happened" (events) from "what to do about it" (reactors) keeps the core logic testable. Rein could use a lightweight version where the monitor emits signals and reactors handle cleanup.
- **Typed error classes** -- Rein's error handling would benefit from structured errors rather than string exceptions, especially for distinguishing "provider crashed" from "user cancelled" from "context limit hit."

**Patterns to avoid:**

- **Full event sourcing** -- the overhead of an append-only event store, projection pipeline, and catch-up replay is not justified for Rein's scope. A simple state object persisted as JSON is sufficient.
- **CQRS separation** -- Rein has one consumer of state (the CLI). Separating read and write models adds complexity without benefit.
- **SQLite migrations** -- Rein should avoid schema migration machinery for its MVP. If persistence is needed, a single JSON file per session is adequate.

---

## 9. Summary

T3Code's orchestration engine is a textbook implementation of event sourcing + CQRS adapted for a coding agent context. The `OrchestrationEngine` handles command dispatch, the decider enforces business rules as pure functions, the `ProjectionPipeline` builds read models, and reactors (`ProviderCommandReactor`, `CheckpointReactor`) drive side effects. SQLite provides durable storage with 13 migrations managing schema evolution. Six typed error classes give structured failure handling, and a formal session state machine prevents invalid transitions.

The architecture is well-suited to T3Code's ambitions as an IDE-integrated tool with rich state management needs. For Rein, the transferable insights are the session state machine, the reactor pattern for side-effect isolation, and structured error typing -- all of which can be adopted in lighter-weight forms without the full event sourcing infrastructure.
