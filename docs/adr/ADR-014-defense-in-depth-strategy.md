# ADR-014: Defense-in-Depth Strategy

## Status

Accepted

## Context

Rein's reliability model relies on multiple independent mechanisms, each catching different failure modes. This strategy emerged organically across several ADRs and core documents but was never named or documented as a deliberate architectural principle. Without an explicit framing, contributors risk bypassing or duplicating layers without understanding what each one catches.

The Principal Skinner deep dive frames safety as a choice between probabilistic controls (prompts) and deterministic controls (sandboxes, kill signals). In practice, Rein uses both — and layers them so that no single mechanism is trusted to guarantee success.

### The problem with single-layer approaches

- **Prompt-only:** The agent can ignore instructions. Hallucinated success, scope creep, and unbounded growth go undetected.
- **Validation-only:** Catches bad output but wastes all tokens before discovering the task was doomed (vague spec, wrong scope).
- **Monitoring-only:** Kills the agent at pressure thresholds but loses all work if nothing was persisted incrementally.

Each mechanism has a failure mode that another mechanism covers. The strategy is to layer them so that **every plausible failure is caught by at least one deterministic control**, with probabilistic controls (prompt instructions) reducing the frequency of failures that reach the deterministic layers.

## Decision

Rein's reliability model is explicitly structured as five layers, ordered from pre-execution to post-failure. Each layer is classified as **probabilistic** (the agent may or may not comply) or **deterministic** (enforced by Rein, not the agent).

### Layer 1 — Pre-Dispatch Specification Validation (Deterministic)

**Catches:** Vague specs, missing validation commands — garbage-in before any tokens are spent.

Runs immediately after task parsing, before any I/O or subprocess work. Two boolean checks: prompt length and validation commands presence. Cheapest possible position in the pipeline.

**Documented in:** [ADR-013](ADR-013-pre-dispatch-specification-validation.md)

### Layer 2 — Structured Prompt Injection (Probabilistic)

**Catches:** Lost work on termination, scope creep, unbounded file growth, missed pre-existing issues.

Rein wraps the operator's task prompt with invisible operational instructions:

- **Commit frequently + stage individually** — persists work incrementally so the wrap-up protocol only captures the last delta.
- **Update PROGRESS.md** — human-readable state that survives agent termination.
- **Read/write LEARNINGS.md** — operational knowledge carries across sessions.
- **Log to DEFERRED.md** — prevents scope creep by giving pre-existing issues an explicit destination.
- **Deviation rules** — 3-tier constraint structure (automatic fixes / scope boundaries / guard rails) that prevents both over-aggressive and over-conservative behavior.
- **Completion signal** — self-verification checklist before writing `.rein/complete`.

This layer is probabilistic — the agent may ignore any instruction. But it is also the cheapest layer (~375 tokens, <0.6% of a 70k budget), and when followed, it dramatically reduces the blast radius of failures caught by later layers.

**Documented in:** [PROMPTS.md](../../PROMPTS.md)

### Layer 3 — Real-Time Context Pressure Monitoring (Deterministic)

**Catches:** Quality degradation from context rot, runaway agents, infinite loops.

Three zones with configurable thresholds:

| Zone | Default | Action |
|------|---------|--------|
| Green | 0–60% | Continue |
| Yellow | 60–80% | Graceful stop (wait for current turn, then kill) |
| Red | >80% | Immediate kill |

Runs concurrently with the agent, parsing the NDJSON/JSONL stream in real-time. Wall-clock timeout runs in parallel — whichever fires first wins.

**Documented in:** [SESSIONS.md](../../SESSIONS.md), [ADR-003](ADR-003-context-pressure-as-primary-metric.md), [ADR-004](ADR-004-zone-based-intervention.md)

### Layer 4 — Quality Gate (Deterministic)

**Catches:** Incorrect output, hallucinated success, incomplete work.

Runs validation commands in the sandbox after agent exit. Binary scoring (all pass → 1.0, any fail → 0.0) with bounded retry rounds (default 4). The completion promise signal ([ADR-002](ADR-002-completion-promise-signal.md)) cross-references the agent's self-assessment against validation results to produce a six-outcome confidence classification: confident, suspicious, overconfident, incomplete, unverified, unevaluated.

**Documented in:** [QUALITY_GATE.md](../../QUALITY_GATE.md), [ADR-002](ADR-002-completion-promise-signal.md)

### Layer 5 — Graceful Recovery (Deterministic)

**Catches:** Data loss from unexpected termination, lost diagnostic context for human continuation.

When an agent is killed (context pressure, timeout, or error):

1. Drain stream buffer and capture partial output
2. Commit uncommitted changes using the agent's git identity
3. Write per-run log with termination metrics
4. Update PROGRESS.md with termination context
5. On terminal failure: generate structured escalation report with branch, commit SHA, diff stat, and progress classification

Learnings extraction runs only after the final verdict — never after intermediate rounds — to avoid propagating false facts from failed attempts.

**Documented in:** [SESSIONS.md § Wrap-Up Protocol](../../SESSIONS.md), [ADR-011](ADR-011-learnings-extraction-after-final-verdict.md), [ADR-012](ADR-012-structured-escalation-report.md)

### Layer Interaction Rules

Three design decisions govern how the layers interact:

1. **The agent does not know about monitoring.** Telling an agent "you might be killed at 60% context usage" causes anxiety-driven shortcuts — rushing through work, producing shallow output, or over-committing early. The commit-frequently instruction (Layer 2) achieves the same crash resilience without the behavioral distortion. This is a deliberate information asymmetry: Layer 2 makes the agent resilient to Layer 3 intervention without the agent knowing Layer 3 exists.

2. **Probabilistic layers reduce load on deterministic layers.** If the agent follows Layer 2 instructions (commits frequently, updates PROGRESS.md), Layer 5's wrap-up protocol only needs to capture the last uncommitted delta — a small, bounded operation. If the agent ignores Layer 2, Layer 5 still works but captures more (potentially inconsistent) state. The layers degrade gracefully when inner layers fail.

3. **Layers are independently disableable.** Each prompt section can be toggled via `AssemblyConfig`. Zone thresholds are configurable per-model. Spec validation has `warn`/`strict`/`skip` modes. This allows operators to benchmark the contribution of each layer and enables debugging when a layer causes unexpected behavior.

### Coverage Matrix

| Failure Mode | L1 (Spec) | L2 (Prompt) | L3 (Monitor) | L4 (Gate) | L5 (Recovery) |
|---|---|---|---|---|---|
| Vague specification | **Primary** | | | | |
| Missing validation commands | **Primary** | | | Meaningless without | |
| Scope creep | | **Primary** | | | |
| Lost work on kill | | **Primary** | | | **Backup** |
| Context rot / quality degradation | | | **Primary** | | |
| Infinite loop / runaway agent | | | **Primary** | | |
| Incorrect output | | | | **Primary** | |
| Hallucinated success | | | | **Primary** | |
| Data loss on crash | | **Primary** | | | **Backup** |
| Diagnostic context for humans | | | | | **Primary** |
| False learnings propagation | | | | | **Primary** |

## What is explicitly not included

- **Runtime layer ordering guarantees.** The layers are conceptual, not a strict execution pipeline. L3 runs concurrently with the agent (not after L2 "completes"). L4 runs after agent exit. L5 runs on failure paths. The ordering in this document reflects when each layer's primary effect occurs, not a sequential execution model.
- **Automated layer health checks.** No mechanism verifies that all layers are active and functioning correctly before dispatch. Operators are responsible for configuration.
- **Cross-layer optimization.** Each layer operates independently. There is no feedback loop where, e.g., L4 results adjust L3 thresholds within a single run. Cross-run optimization (adjusting thresholds based on historical pressure trends) is a future concern.

## Consequences

**Positive:**

- Every plausible failure mode is covered by at least one deterministic control. Prompt instructions (probabilistic) reduce failure frequency but are never the sole defense.
- New contributors can evaluate whether a proposed change maintains defense-in-depth by checking the coverage matrix.
- The information asymmetry decision (agent doesn't know about monitoring) is documented and can be revisited with evidence rather than rediscovered.

**Negative:**

- Five layers create cognitive overhead for contributors who need to understand the full system. This ADR serves as the map.
- Independent disableability means an operator can accidentally remove a layer that another layer depends on (e.g., disabling Layer 2's completion signal makes Layer 4's confidence classification less useful). The coverage matrix shows these dependencies but does not enforce them.

## Source References

- [PROMPTS.md](../../PROMPTS.md) — prompt assembly pipeline and defense-in-depth instructions
- [SESSIONS.md](../../SESSIONS.md) — context pressure monitoring and wrap-up protocol
- [QUALITY_GATE.md](../../QUALITY_GATE.md) — deterministic post-execution validation
- [Principal Skinner Deep Dive — Safety Model](../../research/principal_skinner_deep_dive/01_safety_model.md) — probabilistic vs. deterministic controls framing
- ADRs [002](ADR-002-completion-promise-signal.md), [003](ADR-003-context-pressure-as-primary-metric.md), [004](ADR-004-zone-based-intervention.md), [011](ADR-011-learnings-extraction-after-final-verdict.md), [012](ADR-012-structured-escalation-report.md), [013](ADR-013-pre-dispatch-specification-validation.md) — individual layer implementations
