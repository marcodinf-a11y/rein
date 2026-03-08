# ADR-003: Context Pressure as Primary Metric

## Status

Accepted

## Context

An orchestrator that monitors AI coding agents needs a primary metric to drive intervention decisions — when to let the agent continue, when to stop it gracefully, and when to kill it immediately. Two candidates:

- **Cost (token spend):** How much has been consumed against a budget. Directly maps to dollars. Easy to explain.
- **Context pressure (tokens / context window):** How much of the model's context window has been consumed. Maps to quality degradation risk.

These metrics are correlated but measure different things. A task that uses 50k tokens against a 200k window (25% pressure) is in a different quality state than the same 50k against a 100k window (50% pressure), even though the cost is identical.

Research from Chroma ("Context Rot") demonstrates that model quality degradation is continuous, non-linear, and accelerating as context fills. Larger context windows do not eliminate this — they push the degradation curve further out but preserve the same shape. There is no safe plateau.

## Decision

Context pressure is rein's primary operational metric. Every intervention decision (continue, graceful stop, immediate kill) is driven by context pressure — the ratio of estimated tokens used to the model's context window.

Token budget (default 70k) is tracked as a complementary metric for cost management. It informs reporting and operator awareness but does not trigger intervention. Cost and pressure are both in every report, but pressure drives the control loop.

This means:

- A task can exceed its token budget without being killed, if context pressure is still in the green zone (e.g., large context window model).
- A task can be killed well under budget, if context pressure enters yellow/red (e.g., small context window model).
- The operator sees both metrics in the report and can adjust budgets or zone thresholds independently.

## Consequences

**Positive:**

- Intervention is tied to the actual failure mode (quality degradation), not a proxy (cost).
- Models with different context windows get appropriate treatment automatically — a 1M-window model has more room than a 128k model, and pressure reflects this.
- Cost tracking remains available for resource planning without conflating it with quality risk.

**Negative:**

- Context pressure requires knowing the model's context window size. Not all agents report it at invocation time (only Claude provides `contextWindow` in `modelUsage`). For others, rein uses a model metadata lookup.
- Operators accustomed to thinking in dollar terms must learn a second metric.
- When mid-run token monitoring is unavailable (Gemini, Claude with extended thinking), pressure can only be computed post-completion — rein cannot intervene mid-run in degraded mode.
