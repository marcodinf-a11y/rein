# Enrichment Proposals: Overview & Rationale

**March 2026**

Proposals for enriching rein design docs with patterns identified in the Gas Town deep dive. Each proposal is self-contained and targets a specific rein doc. No existing files are modified — these are proposed additions for review.

---

## Proposal Index

| # | Proposal | Target Doc | Priority | Source |
|---|----------|-----------|----------|--------|
| [01](01_gas_town_ecosystem_positioning.md) | Gas Town Ecosystem Positioning | ARCHITECTURE.md | Now | [Gas Town synthesis](../00_synthesis.md) §2, §3 |
| [02](02_crash_recoverable_task_state.md) | Crash-Recoverable Task State | TASKS.md | Now | [Gas Town architecture](../01_architecture.md) §4, [coordination](../02_coordination.md) §2 |
| [03](03_specification_validation.md) | Pre-Dispatch Specification Validation | TASKS.md | Now | [Gas Town task decomposition](../04_task_decomposition.md) §2, §6 |
| [04](04_merge_queue.md) | Merge Queue for Concurrent Agents | SESSIONS.md | Next | [Gas Town coordination](../02_coordination.md) §3, §7 |
| [05](05_workflow_persistence.md) | Workflow Persistence (Molecule Pattern) | TASKS.md | Next | [Gas Town architecture](../01_architecture.md) §4, [critical analysis](../05_critical_analysis.md) §5 |
| [06](06_reusable_task_templates.md) | Reusable Task Templates (Formula Pattern) | TASKS.md | Next | [Gas Town task decomposition](../04_task_decomposition.md) §3, §6 |

---

## Decision Rationale

### Why These Six

Each proposal addresses a gap where Gas Town has a demonstrated pattern that rein lacks:

1. **Gas Town Ecosystem Positioning** — Rein docs reference Ralph and ralph-orchestrator but not Gas Town, the most prominent multi-agent orchestration system (11K+ stars). Documenting the relationship clarifies where rein sits in the concurrent-vs-sequential spectrum.

2. **Crash-Recoverable Task State** — Gas Town's Beads (JSONL in git) ensure tasks survive crashes, context fills, and restarts. Rein's task artifacts should be similarly crash-recoverable. This is the most directly transferable pattern.

3. **Pre-Dispatch Specification Validation** — Gas Town's biggest operational lesson: vague specs waste compute at scale. A lightweight pre-dispatch check catches the most common failure mode before any tokens are spent.

4. **Merge Queue** — When rein eventually supports concurrent agents, merge management will be the hardest problem. Gas Town's Refinery pattern provides a concrete, tested model.

5. **Workflow Persistence** — Gas Town's Molecules (durable workflow DAGs with checkpoints) enable crash-recoverable multi-step workflows. Rein needs a similar concept for multi-task sequences.

6. **Reusable Task Templates** — Gas Town's Formulas (TOML workflow templates) reduce the overhead of defining common workflows. Rein should offer predefined task sequences for common operations.

### What Was Rejected (and Why)

These patterns from Gas Town were evaluated and rejected:

| Pattern | Why Rejected | Source |
|---------|-------------|--------|
| **MEOW stack (5-layer abstraction)** | Over-engineered. Beads → Epics → Molecules → Protomolecules → Formulas adds cognitive overhead without proportional value. Two levels (task group → task) is sufficient. | [critical analysis](../05_critical_analysis.md) §6 |
| **20-30 concurrent agents** | Unproven cost/quality tradeoff. Super-linear cost scaling, $2-5K/month waste reported, no quality benchmarks. | [resource management](../03_resource_management.md) §2, §3 |
| **LLM-as-supervisor (Witness/Deacon)** | Consumes 15-30% of token budget for monitoring without structured quality gates. Rein's deterministic binary evaluation is cheaper and more reliable. | [resource management](../03_resource_management.md) §3.2 |
| **Impenetrable naming (GUPP, Polecats, Wisps)** | Optimizes for one person's mental model, not usability. Use clear, descriptive names. | [critical analysis](../05_critical_analysis.md) §6 |
| **No cost tracking** | Gas Town's explicit lack of per-agent cost tracking is its biggest operational gap. Rein already has normalized token accounting — never drop this. | [resource management](../03_resource_management.md) §2, §4 |
| **LLM-only merge resolution** | Gas Town's Refinery resolves merge conflicts via LLM without structured verification. Rein should add deterministic verification (tests) after any LLM merge. | [coordination](../02_coordination.md) §3 |

### Mapping to Synthesis Tiers

| Tier | Proposals |
|------|-----------|
| **Now** (low effort, no new infrastructure) | 01 (positioning), 02 (crash-recoverable state), 03 (spec validation) |
| **Next** (when multi-task mode is built) | 04 (merge queue), 05 (workflow persistence), 06 (task templates) |
| **Never** | MEOW complexity, 20-30 agents, LLM-as-supervisor, impenetrable naming, drop cost tracking, LLM-only merge |

---

## Relationship to Ralph Orchestrator Proposals

The ralph-orchestrator proposals (`research/ralph_orchestrator_deep_dive/proposals/`) address iteration-level patterns (stagnation detection, inter-session validation, completion promise). The Gas Town proposals address coordination-level patterns (crash recovery, merge queues, workflow persistence). They are complementary:

| Scope | Ralph Proposals | Gas Town Proposals |
|-------|----------------|-------------------|
| **Single session** | Completion promise, LEARNINGS.md | — |
| **Between sessions** | Stagnation detection, inter-session validation, event stream | Crash-recoverable state, spec validation |
| **Multi-agent** | — | Merge queue, workflow persistence, task templates |
| **Documentation** | Ralph ecosystem positioning | Gas Town ecosystem positioning |

---

## Sources

- [Gas Town Deep Dive: Synthesis](../00_synthesis.md)
- [Gas Town: Architecture](../01_architecture.md)
- [Gas Town: Coordination](../02_coordination.md)
- [Gas Town: Resource Management](../03_resource_management.md)
- [Gas Town: Task Decomposition](../04_task_decomposition.md)
- [Gas Town: Critical Analysis](../05_critical_analysis.md)
- [Ralph Orchestrator Proposals](../../ralph_orchestrator_deep_dive/proposals/00_overview.md)
- Rein design docs: ARCHITECTURE.md, SESSIONS.md, TASKS.md, REPORTS.md
