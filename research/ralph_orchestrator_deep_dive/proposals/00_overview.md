# Enrichment Proposals: Overview & Rationale

**March 2026**

Proposals for enriching rein design docs with patterns identified in the ralph-orchestrator and Ralph Wiggum deep dives. Each proposal is self-contained and targets a specific rein doc. No existing files are modified — these are proposed additions for review.

---

## Proposal Index

| # | Proposal | Target Doc | Priority | Source |
|---|----------|-----------|----------|--------|
| [01](01_ralph_ecosystem_positioning.md) | Ralph Ecosystem Positioning | ARCHITECTURE.md | Now | [ralph-orchestrator synthesis](../00_synthesis.md) §2, [Ralph Wiggum synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) §3 |
| [02](02_stagnation_detection.md) | Stagnation Detection | SESSIONS.md | Next | [ralph-orchestrator synthesis](../00_synthesis.md) §3.1, [iteration control](../03_iteration_control.md) §3 |
| [03](03_inter_session_validation.md) | Inter-Session Validation | SESSIONS.md | Next | [ralph-orchestrator synthesis](../00_synthesis.md) §3.2, [iteration control](../03_iteration_control.md) §5 |
| [04](04_completion_promise.md) | Completion Promise Signal | TASKS.md | Now | [Ralph Wiggum synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) §7.2, [iteration control](../03_iteration_control.md) §2 |
| [05](05_learnings_seed_file.md) | LEARNINGS.md Seed File | TASKS.md | Now | [Ralph Wiggum synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) §7.3 |
| [06](06_event_stream.md) | JSONL Event Stream | REPORTS.md | Next | [ralph-orchestrator synthesis](../00_synthesis.md) §3.3, [evaluation & reporting](../04_evaluation_reporting.md) §7 |

---

## Decision Rationale

### Why These Six

Each proposal addresses a gap where ralph-orchestrator or bare Ralph has a mature, tested pattern that rein lacks:

1. **Ralph Ecosystem Positioning** — Rein docs don't mention Ralph anywhere. Given that ralph-orchestrator is the closest community implementation (2,088 stars, same core insight of context rotation), documenting the relationship clarifies Rein's unique value proposition.

2. **Stagnation Detection** — Ralph's three-detector model (stale, thrashing, consecutive failures) is the most mature stagnation detection in the ecosystem. Rein lists this as "planned" but has no design. Ralph provides a concrete, tested model to adapt.

3. **Inter-Session Validation** — Ralph's backpressure gates (validation between iterations) catch regressions early. Rein only validates after the final session. For multi-task workflows, early failure detection reduces wasted compute.

4. **Completion Promise** — Low-effort addition that adds a confidence dimension to the quality gate. Ralph's `ralph emit LOOP_COMPLETE` with required-events validation is a proven pattern.

5. **LEARNINGS.md Seed File** — Formalizes operational knowledge transfer between sessions. Ralph's scratchpad serves this purpose; rein has seed files but no pattern for knowledge (vs. code) transfer.

6. **JSONL Event Stream** — Ralph's event bus provides more granularity than Rein's before/after snapshot model. A continuous event log enables debugging, trend analysis, and CI/CD integration.

### What Was Rejected (and Why)

These patterns from ralph-orchestrator were evaluated and rejected:

| Pattern | Why Rejected | Source |
|---------|-------------|--------|
| **Hat system** (persona routing) | Rein monitors agents, it doesn't orchestrate personas. Adding persona routing would conflate monitoring with orchestration. | [synthesis](../00_synthesis.md) §4.3 |
| **PTY execution** | PTY introduces terminal escape codes that complicate stream parsing. Piped stdout is more reliable for Rein's monitoring mission. | [synthesis](../00_synthesis.md) §4.4 |
| **Markdown reports** | Machine-readable JSON is non-negotiable for systematic evaluation. Markdown summaries are human-readable but not programmatically comparable. | [synthesis](../00_synthesis.md) §4.2 |
| **Drop context pressure monitoring** | Ralph's "short iterations avoid the problem" is a heuristic, not a solution. The zone model is Rein's core value. | [synthesis](../00_synthesis.md) §4.1 |
| **Scratchpad (16K FIFO)** | Rein's seed files + LEARNINGS.md achieves the same goal with more operator control. | [synthesis](../00_synthesis.md) §3.4 |

### Mapping to Synthesis Tiers

| Tier | Proposals |
|------|-----------|
| **Now** (low effort, no new infrastructure) | 01 (positioning), 04 (completion promise), 05 (LEARNINGS.md) |
| **Next** (when multi-task mode is built) | 02 (stagnation detection), 03 (inter-session validation), 06 (event stream) |
| **Never** | Hat system, PTY execution, markdown reports, drop context pressure |

The "Now" tier items from the [Ralph Wiggum synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md) §8 and [ralph-orchestrator synthesis](../00_synthesis.md) §5 are all represented. The "Next" tier items that depend on multi-task mode are scoped here as designs ready for implementation when that prerequisite lands.

---

## Sources

- [Ralph Orchestrator Deep Dive: Synthesis](../00_synthesis.md)
- [Ralph Wiggum Deep Dive: Synthesis](../../ralph_wiggum_deep_dive/00_synthesis.md)
- [Iteration Control](../03_iteration_control.md)
- [Evaluation & Reporting](../04_evaluation_reporting.md)
- Rein design docs: ARCHITECTURE.md, SESSIONS.md, TASKS.md, REPORTS.md
