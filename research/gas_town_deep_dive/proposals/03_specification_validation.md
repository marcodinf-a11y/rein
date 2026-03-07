# Proposal: Pre-Dispatch Specification Validation

**Target:** TASKS.md (task dispatch enhancement)
**Priority:** Now
**Effort:** Low

---

## Proposed Enhancement: Specification Validation Gate

The following describes a proposed enhancement to the task dispatch pipeline, documented in TASKS.md.

---

### Pre-Dispatch Specification Validation

Before dispatching a task to an agent, rein validates that the task definition is specific enough for reliable execution. This catches the most common failure mode identified across Gas Town, Goosetown, and Ralph Loop deployments: vague specifications that waste compute.

#### Validation Checks

| Check | Validates | Failure Mode Prevented |
|-------|----------|----------------------|
| **Description length** | `description` has >= 50 characters | One-liner descriptions ("fix the auth") produce ambiguous agent behavior |
| **Validation commands present** | `validation_commands` is non-empty | Tasks without validation cannot be evaluated — the quality gate is meaningless |
| **Acceptance criteria** | `description` contains testable criteria (checked heuristically) | Vague goals ("make it better") produce unfalsifiable outcomes |

#### Behavior

```
TaskDefinition → Spec Validation → pass? → Dispatch to Agent
                                    │
                                    └─ fail → Warning + operator prompt
```

- **Default:** Warn on validation failure but dispatch anyway (operator override)
- **Strict mode** (`--strict-spec`): Block dispatch on validation failure
- **Skip mode** (`--no-spec-check`): Disable validation entirely

#### Warning Output

```
Warning: Task "refactor-auth" has specification issues:
  - No validation commands defined (quality gate will be meaningless)
  - Description is 23 characters (recommend >= 50 for reliable execution)

Dispatch anyway? [y/N]
```

#### Not Included

This proposal explicitly does **not** include:
- LLM-based specification analysis (Gas Town's Mayor decomposition). Validation is deterministic and fast.
- Automatic spec improvement (rewriting the task description). The operator writes specs.
- Semantic validation (checking if the description makes sense). Only structural checks.

---

## Rationale

Gas Town's most consistent operational lesson, reported by multiple independent observers:

- "The limiting factor is how fast you can think, give direction, and validate" — [Embracing Enigmas](https://embracingenigmas.substack.com/p/exploring-gas-town)
- "Planning and design become the limiting factor. Humans must deeply consider what to build before prompting agents" — [Maggie Appleton](https://maggieappleton.com/gastown)

Gas Town's Mayor performs model-driven decomposition — the LLM decides how to break down vague prompts. This amplifies vague specifications rather than catching them: a vague top-level prompt produces vague Beads, each of which produces vague agent behavior, compounding waste across 20-30 concurrent agents.

Rein uses operator-defined task sequences, which avoids the amplification problem but doesn't prevent the root cause: operators writing vague tasks. A lightweight pre-dispatch gate catches this before any tokens are spent.

The checks are deliberately simple — character count, field presence, basic heuristics. This avoids the trap of using an LLM to validate specs (which would cost tokens and introduce its own failure modes). The goal is not to ensure spec quality — that's the operator's responsibility — but to catch obvious omissions (no validation commands, one-word descriptions) that guarantee wasted execution.

Stripe Minions solves this differently: blueprints are structured templates with mandatory fields, making vague specs impossible by construction. Rein's approach is lighter-weight (warnings, not mandatory structure) but achieves the same early detection of the most common failure mode.

## Source References

- [Gas Town Task Decomposition](../04_task_decomposition.md) — Section 2 (specification quality as bottleneck), Section 6 (implications)
- [Gas Town Critical Analysis](../05_critical_analysis.md) — Section 3 (evidence against concurrent multi-agent without good specs)
- [Gas Town Resource Management](../03_resource_management.md) — Section 3.3 (wasted work from vague specs)
- TASKS.md — current TaskDefinition schema (no validation of spec quality)
