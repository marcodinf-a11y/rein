# Proposal: Orchestrator Budget Ceiling

**Priority:** Now | **Effort:** Low | **Source:** GSD Deep Dive

---

## Summary

Document the architectural principle that orchestrator overhead must never consume significant context. Rein naturally achieves this (external Python process), but the principle should be explicit to prevent future scope creep.

## Rationale

GSD's "lean orchestrator" concept (~10-15% of context window) is sound but unenforced — it's a prompt comment, not a runtime constraint. GSD's orchestrator runs inside Claude Code's context window, so it competes with work context and is subject to the same degradation it's trying to prevent.

Rein's external architecture is superior: the orchestrator (Python CLI) is entirely outside the agent's context window. Zero context overhead. This is a deliberate and valuable architectural property that should be documented as a design invariant.

## Proposed Content

Add to ARCHITECTURE.md under the "System Overview" or "Why Python" section:

### Orchestrator Context Overhead: Zero by Design

Rein orchestrator is an external Python process. It does not consume any tokens in the agent's context window. This is a deliberate architectural choice:

- **GSD's approach:** Orchestrator runs inside Claude Code's session (~10-15% context, unmeasured, unenforced)
- **Gas Town's approach:** Mayor/Witness supervisors run as agents (~15-30% token waste per supervised session)
- **Rein approach:** Orchestrator is external (Python subprocess management). Zero context overhead. The agent's full context window is available for work.

This property must be preserved. Rein should never move orchestration logic into the agent's context window.

### Invariant

```
Agent context window = 100% available for task work
Orchestrator context = 0% (external process)
```

If future features require in-session coordination (e.g., multi-turn workflows with `--resume`), the coordination messages should be minimal and their context cost should be tracked as orchestrator overhead.

## Why Document This Now

Rein achieves this property naturally today. Documenting it as an invariant prevents future decisions that would compromise it:
- Moving task orchestration into CLAUDE.md instructions (GSD's pattern)
- Adding "supervisor agents" that consume context (Gas Town's pattern)
- Using `--resume` with extensive prior conversation context

## Target Document

ARCHITECTURE.md — add as a design invariant, near the system overview

## Verification

- [ ] Invariant is documented in ARCHITECTURE.md
- [ ] No rein code consumes agent context tokens
- [ ] Future multi-turn support tracks orchestrator context cost
