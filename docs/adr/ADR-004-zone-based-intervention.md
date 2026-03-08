# ADR-004: Zone-Based Intervention Model

## Status

Accepted

## Context

Given that context pressure drives intervention (ADR-003), rein needs a policy for when to act. Two approaches:

- **Continuous curve:** Track pressure as a float and apply graduated responses (e.g., reduce tool availability at 50%, inject "hurry up" prompts at 70%, kill at 90%). Requires complex heuristics and calibration per model.
- **Discrete zones:** Define threshold boundaries that partition the pressure range into named zones, each with a single clear action. Operationally simple — operators configure two numbers and get deterministic behavior.

The gap between zones matters. If the yellow-to-red gap is too small, the agent can push from green into red within a single turn (a large tool call can consume 10-20% of context), bypassing yellow entirely.

## Decision

Three discrete pressure zones with configurable boundaries:

| Zone | Default Range | Action |
|------|--------------|--------|
| Green | 0–60% | Continue execution |
| Yellow | 60–80% | Graceful stop — wait for current turn to complete, then kill |
| Red | >80% | Immediate kill |

Zone boundaries are configurable globally and per-model via `context_pressure` config. The recommended minimum gap between green and red is 15–20% of the context window to ensure the agent cannot skip yellow in a single turn. Rein does not enforce a minimum gap — this is the operator's responsibility.

## Consequences

**Positive:**

- Deterministic and auditable. Given a pressure reading, the action is unambiguous.
- Operators can tune thresholds per model based on empirical degradation curves (e.g., a model known to degrade early gets tighter boundaries).
- Yellow zone provides a "landing strip" — the agent finishes its current turn before being stopped, avoiding half-completed tool calls that corrupt sandbox state.

**Negative:**

- Discrete zones cannot express graduated responses. There is no "slow down" signal — only "continue" and "stop."
- A single-turn token spike can jump from green to red, bypassing yellow. The operator must size the gap appropriately for their workload.
- Three zones may be too coarse. A future "orange" zone (e.g., inject wrap-up instructions) could add nuance, but is not implemented.
