# ADR-009: Protocol Over ABC for Agent Adapters

## Status

Accepted

## Context

Rein needs an interface contract for agent adapters — each adapter (Claude, Codex, Gemini) must expose the same shape so the orchestrator can invoke any agent uniformly. Python offers two mechanisms:

- **Abstract Base Class (ABC):** Adapters inherit from a base class with `@abstractmethod` declarations. Enforces the contract at class definition time. Couples all adapters to the base class import.
- **Protocol (PEP 544):** Structural typing. Adapters match a shape without inheriting anything. The contract is checked at type-check time (mypy) or via `runtime_checkable`. No import coupling.

## Decision

The `AgentAdapter` interface is a `typing.Protocol` with `@runtime_checkable`:

```python
@runtime_checkable
class AgentAdapter(Protocol):
    @property
    def name(self) -> str: ...

    def is_available(self) -> bool: ...

    async def run(self, task: TaskDefinition, sandbox_path: Path) -> AgentResult: ...
```

Adapters do not inherit from `AgentAdapter`. They satisfy the protocol by implementing the required attributes and methods with matching signatures.

## Consequences

**Positive:**

- Adapters are decoupled. `claude.py` does not import `base.py` — it just matches the shape. This matters when adding external or third-party adapters.
- No base class constructor to call, no `super().__init__()` chains.
- `runtime_checkable` allows `isinstance(adapter, AgentAdapter)` checks at runtime for validation.
- Consistent with Python's "duck typing" philosophy — if it quacks like an adapter, it is one.

**Negative:**

- Protocol violations are caught at type-check time (mypy) or runtime (`isinstance`), not at class definition time. A malformed adapter that is never type-checked or instantiated will not raise an error.
- Cannot provide default implementations. With ABC, a base class could implement `is_available()` as `shutil.which(self.binary_name) is not None`. With Protocol, each adapter implements this independently.
- Less familiar to developers coming from Java/C# patterns where interface inheritance is the norm.
