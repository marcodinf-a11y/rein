# Gas Town: Resource & Cost Management

**March 2026**

---

## 1. The Cost Problem at Scale

Running 20-30 concurrent AI coding agents is expensive. Gas Town is remarkably transparent about this — and remarkably lacking in cost management tooling.

Yegge's own guidance: "You won't like Gas Town if you ever have to think...about where money comes from." This is both honest and concerning.

Source: [Torq Reading List](https://reading.torqsoftware.com/notes/software/ai-ml/agentic-coding/2026-01-15-gas-town-multi-agent-orchestration-framework/)

---

## 2. Token Usage Tracking

**Gas Town does not track token usage across agents.** There is no per-agent token accounting, no aggregate cost dashboard, no real-time spend visibility. The system relies on the underlying LLM provider's billing (Anthropic API usage, Claude Pro/Max subscriptions).

This is Gas Town's most significant operational gap. Users report:

- "You can burn through half of a Pro Max account (or several) in six to eight hours running hot" — [Embracing Enigmas](https://embracingenigmas.substack.com/p/exploring-gas-town)
- "$2,000-5,000 monthly through inefficiency — lost work, redundant bug fixes, repeated designs" — [Maggie Appleton](https://maggieappleton.com/gastown)

Without per-agent cost tracking, operators cannot identify which agents or tasks are consuming disproportionate resources.

---

## 3. Cost Sources and Waste

Gas Town's cost profile has several components:

### 3.1 Direct Agent Execution
Each Polecat session consumes tokens proportional to:
- Context loading (reading codebase, Beads state, role instructions)
- Task execution (code generation, tool calls)
- Verification (running tests, checking results)

### 3.2 Supervision Overhead
The Witness and Deacon roles consume tokens purely for monitoring:
- Witness periodically "nudges" workers to check progress
- Deacon monitors system health
- Dogs handle maintenance tasks

These supervision agents can consume 15-30% of total token budget without producing code. Multiple observers flag this as "watchers watching watchers" — a cost sink with unclear ROI.

### 3.3 Wasted Work
The most expensive cost is **wasted work from merge conflicts**:
- Two agents modify the same area → one agent's work is partially discarded
- Refinery resolution may introduce bugs requiring rework
- Failed merge attempts consume additional tokens for retry/reimplementation

### 3.4 Context Reload
Every time a Polecat spawns, it must reload context (codebase, task definition, role instructions). This "cold start" cost is multiplied across dozens of ephemeral agents.

---

## 4. Per-Agent Cost Caps

**Gas Town has no per-agent cost caps or budgets.** There is no mechanism to:
- Set a maximum token budget per Polecat session
- Limit the number of tool calls per agent
- Kill a runaway agent based on cost thresholds
- Compare per-task cost against expected cost

This contrasts sharply with Rein's design, which includes per-task token budgets and normalized accounting across different agent backends.

---

## 5. Runaway Agent Handling

Gas Town handles runaway agents through two mechanisms:

### 5.1 Witness Supervision
The Witness monitors worker progress and can intervene if an agent appears stalled or off-track. However, this is an LLM making judgment calls about another LLM's behavior — a noisy and unreliable signal.

### 5.2 Context Window Exhaustion
When an agent fills its context window, the session crashes. Gas Town treats this as normal operation — the GUPP principle ensures the agent's work persists in Beads and can be resumed by a fresh session. This is "crash as cost control" — a brute-force mechanism, not a designed one.

Neither mechanism is a proper cost control. There is no kill switch based on token spend.

---

## 6. Comparison: Cost Management Approaches

| System | Token Tracking | Cost Caps | Runaway Protection | Cost Visibility |
|--------|---------------|-----------|--------------------|-----------------|
| **Gas Town** | None | None | Witness + context crash | Provider billing only |
| **Stripe Minions** | Per-minion tracking | Blueprint-scoped budgets | Blueprint timeout | Internal dashboards |
| **Ralph Orchestrator** | None | None | Stagnation detection (3 detectors) | JSONL event logging |
| **Swarm-Tools** | Checkpoint-based (25/50/75%) | Configurable | Progress milestone gates | Event sourcing log |
| **Rein (planned)** | Normalized per-task accounting | Per-task token budgets | Zone-based intervention (green/yellow/red) | Structured JSON metrics |

Source: `research/stripe_deep_dive/`, `research/ralph_orchestrator_deep_dive/`, [Swarm-Tools comparison](https://gist.github.com/johnlindquist/4174127de90e1734d58fce64c6b52b62)

---

## 7. Implications for Rein

### Rein is already ahead on cost management
Gas Town's lack of cost tracking validates Rein's design decision to include normalized token accounting from the start. Per-task token budgets, real-time monitoring, and zone-based intervention are exactly what Gas Town users wish they had.

### Cost scales super-linearly with agent count
Gas Town demonstrates that costs don't scale linearly with agents — supervision overhead, wasted work from conflicts, and context reload costs create super-linear scaling. Rein should model this explicitly if it ever supports concurrent agents.

### Supervision agents are expensive
Gas Town's Witness/Deacon pattern (LLM supervising LLM) consumes 15-30% of token budget for monitoring. Rein's approach — structured evaluation with binary scoring — is dramatically cheaper and more deterministic. Never adopt LLM-as-supervisor for cost-sensitive operations.

### Context reload is a hidden cost
Every ephemeral agent session pays a "cold start" tax to reload context. Rein should track and optimize this cost, potentially through context caching or pre-compiled context snapshots for recurring task patterns.
