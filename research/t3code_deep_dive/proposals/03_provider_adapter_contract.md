# Proposal: Provider Adapter Contract Enhancements

**Target:** ARCHITECTURE.md (modify "Agent Adapter Interface" section)
**Priority:** P1 (R1)
**Effort:** Low

---

## Proposed Section: Enhanced AgentAdapter Protocol

The following is a proposed replacement for the "Agent Adapter Interface" section in ARCHITECTURE.md.

---

### Agent Adapter Interface

Uses Python `Protocol` (structural typing). Adapters don't inherit from a base class — they match the shape:

```python
@dataclass(frozen=True)
class AdapterHealth:
    """Result of a pre-flight health check."""
    available: bool          # CLI binary found and callable
    version: str | None      # CLI version string, if detectable
    error: str | None        # Error message if not available

@runtime_checkable
class AgentAdapter(Protocol):
    """Protocol all agent adapters must satisfy."""

    @property
    def name(self) -> str:
        """Human-readable agent name, e.g. 'claude-code'."""
        ...

    def is_available(self) -> bool:
        """Check if the agent CLI is installed and callable."""
        ...

    def check_health(self) -> AdapterHealth:
        """Pre-flight health check with structured result.

        Returns version info when available. Used by `rein list agents`
        and as a pre-dispatch guard.
        """
        ...

    async def run(
        self, task: TaskDefinition, sandbox_path: Path
    ) -> AgentResult:
        """Execute the task in the given sandbox directory."""
        ...

    def stream_events(
        self, process: asyncio.subprocess.Process
    ) -> AsyncIterator[AgentStreamEvent]:
        """Parse the agent's stdout stream into typed events.

        Each adapter implements its own parsing logic:
        - Claude: NDJSON message_start/message_delta/result events
        - Codex: JSONL turn.started/turn.completed/item.completed events
        - Gemini: NDJSON message/tool_use/result events

        The monitor consumes this iterator to compute context pressure.
        """
        ...
```

The `run` method is `async` because all three CLIs are subprocess invocations that benefit from `asyncio.create_subprocess_exec`.

The `stream_events` method decouples stream parsing from monitoring logic. Each adapter translates its agent's output format into a common `AgentStreamEvent` type, which the pressure monitor consumes. This means adding a new agent requires only implementing its parsing logic — the monitor does not change.

`check_health` extends `is_available` with structured results. `is_available` returns a boolean for quick checks; `check_health` returns version info and error details for `rein list agents` and diagnostic output.

---

## Rationale

T3Code's `ProviderAdapterShape<E>` interface defines 11 methods including a `streamEvents: Stream<ProviderRuntimeEvent>` — the core observation channel. The key insight: the adapter should emit typed events, not raw stdout lines. This decouples monitoring from parsing.

Rein's current `AgentAdapter` protocol has `is_available()` and `run()`. The monitor parses stdout directly, which means monitoring logic must understand each agent's output format. Adding `stream_events()` to the adapter protocol pushes format-specific parsing into the adapter where it belongs.

Two T3Code adapter methods are adopted:
- **`stream_events()`** — maps to T3Code's `streamEvents` property. Returns typed events instead of raw lines.
- **`check_health()`** — maps to T3Code's `getHealth()`. Structured health check for diagnostics.

T3Code adapter methods explicitly skipped:
- `interrupt()` / `resume()` — Rein uses subprocess signals (SIGTERM/SIGINT), not adapter-level protocol. Defer to R2 when session resume is added.
- `rollback(checkpoint)` — requires event sourcing infrastructure Rein doesn't have.
- `sendMessage()` — Rein doesn't inject messages into running sessions.
- `getModels()` — model configuration is in `rein.toml`, not queried from adapters.

## Source References

- [T3Code Deep Dive: Provider Adapter Pattern](../03_provider_adapter_pattern.md) — `ProviderAdapterShape<E>`, 11 methods, `streamEvents`
- [T3Code Deep Dive: Synthesis](../00_synthesis.md) — §3 (adapter pattern well-designed)
- [T3Code Deep Dive: Critical Analysis](../07_critical_analysis.md) — §4 (Rein should adopt stream-based observation)
- ARCHITECTURE.md — current `AgentAdapter` protocol (3 methods: `name`, `is_available`, `run`)
