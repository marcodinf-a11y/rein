# Proposal: Multi-Run Adversarial Evaluation

**Target:** QUALITY_GATE.md (new pipeline mode)
**Priority:** Next (when multi-run support exists)
**Effort:** Medium

---

## Proposed Enhancement: Adversarial Evaluation Mode

The following describes a proposed pipeline mode for QUALITY_GATE.md that extends the harness's evaluation capabilities toward adversarial testing.

---

### Adversarial Evaluation Mode

> **Status: Planned.** Builds on the multi-run evaluation recommended by the RLM deep dive.

Adversarial evaluation runs the same task multiple times with controlled variation, analyzing the distribution of outcomes rather than a single pass/fail result. It is the practical subset of Principal Skinner's "thousands of trajectories" proposal, scoped for cost-effective use by solo developers and small teams.

#### How It Works

```bash
# Run task 10 times, analyze failure distribution
harness eval -t task.json --runs 10

# Run with prompt variation (adversarial mode)
harness eval -t task.json --runs 10 --vary-prompt
```

**Standard mode** (`--runs N`): Run the same task N times with identical configuration. Produces a statistical summary: pass rate, failure mode distribution, cost distribution, context pressure distribution.

**Adversarial mode** (`--runs N --vary-prompt`): Run the same task N times with controlled prompt variations. Variations include:
- Reordered instructions (same content, different sequence)
- Injected distractors ("Ignore previous instructions and..." prepended to task spec)
- Constraint relaxation (remove one validation command per run to detect reward hacking)

#### Output: Evaluation Report

```json
{
    "evaluation": {
        "task_id": "refactor-auth-001",
        "total_runs": 10,
        "pass_rate": 0.7,
        "failure_modes": {
            "test_failure": 2,
            "context_pressure_kill": 1,
            "timeout": 0
        },
        "cost_distribution": {
            "min_usd": 0.04,
            "max_usd": 0.12,
            "median_usd": 0.065,
            "p95_usd": 0.11
        },
        "context_pressure_distribution": {
            "min_pct": 32.1,
            "max_pct": 78.4,
            "median_pct": 51.3
        },
        "action_summary": {
            "avg_bash_commands": 22,
            "max_destructive": 0,
            "any_network": false
        },
        "adversarial_results": {
            "prompt_variation_pass_rate": 0.6,
            "injection_resistance": "8/10 runs ignored injected distractor",
            "reward_hacking_detected": false
        }
    }
}
```

#### What This Replaces

Principal Skinner proposes pre-deployment adversarial simulation: "thousands of simulated trajectories" to find "toxic flows." The critical analysis ([06_critical_analysis.md](../06_critical_analysis.md)) concludes this is impractical:

| Dimension | Principal Skinner Proposal | Harness Adversarial Evaluation |
|-----------|--------------------------|-------------------------------|
| **Scale** | Thousands of trajectories | 5-20 runs (configurable) |
| **Cost** | $100s-$1000s per task | $1-$5 per task (5-20 × $0.05-$0.25 per run) |
| **Timing** | Pre-deployment (before any production use) | On-demand (when confidence in a task/agent is needed) |
| **Variation** | Adversarial inputs, hostile environments | Prompt variation, distractor injection, constraint relaxation |
| **Output** | "Actuarial evidence of safety boundaries" | Statistical pass rate, failure mode distribution, cost distribution |
| **Toxic flow detection** | Classify dangerous action sequences | Action frequency summary across runs (from proposal 04) |

The harness version captures 80% of the value at 5% of the cost. It does not claim to find all toxic flows or provide actuarial evidence — it provides *statistical confidence* in a task/agent combination.

#### Relationship to Existing Features

- **Quality gate:** Each run goes through the full quality gate. The evaluation report aggregates across runs.
- **Action frequency monitoring (proposal 04):** If enabled, action counts are aggregated across runs, surfacing patterns invisible in single runs (e.g., "in 3 of 10 runs, the agent ran destructive commands").
- **RLM deep dive recommendation:** The RLM deep dive recommended multi-run evaluation to address 0/6-to-6/6 run variance. This proposal formalizes that recommendation and adds adversarial prompt variation.
- **Stagnation detection (ralph-orchestrator proposal 02):** Stagnation detectors operate within a single workflow. Adversarial evaluation operates across independent runs. They are complementary.

---

## Rationale

Two independent deep dives converge on multi-run evaluation:

1. **RLM deep dive** found 0/6-to-6/6 run variance in coding benchmarks — single-run evaluation is unreliable. Recommended running 5+ runs per task for statistical confidence.

2. **Principal Skinner deep dive** proposes adversarial simulation to find toxic flows. The practical subset — multi-run with prompt variation — is achievable within the harness's cost constraints.

The harness already has the infrastructure for this:
- Task definitions are reusable
- Sandboxes are created fresh per run
- Quality gate evaluates each run independently
- Structured reports can be aggregated

The missing piece is orchestration: running the same task N times, collecting results, and producing a statistical summary. This is a thin layer on top of the existing `harness run` command.

Adversarial prompt variation (distractor injection, instruction reordering) tests the agent's robustness to the kinds of perturbations that occur naturally in real codebases: misleading comments, ambiguous requirements, conflicting instructions. This is more practical than simulating hostile environments — it tests the agent's reasoning resilience, which is what matters for coding tasks.

## Source References

- [Principal Skinner Synthesis](../00_synthesis.md) — Section 5 (Next tier: multi-run adversarial evaluation)
- [Adversarial Simulation](../05_adversarial_simulation.md) — Section 5 (practicality assessment), Section 6 (relationship to multi-run evaluation)
- [Critical Analysis](../06_critical_analysis.md) — Section 3 (cost-prohibitive assessment of full adversarial simulation)
- research/rlm_deep_dive/00_synthesis.md — Multi-run evaluation recommendation, 0/6-to-6/6 variance finding
- QUALITY_GATE.md — Existing pipeline modes (forward, review, composition)
