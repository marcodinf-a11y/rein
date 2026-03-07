# Proposal: Ecosystem Positioning — GSD Comparison

**Priority:** Now | **Effort:** Low | **Source:** GSD Deep Dive

---

## Summary

Add a section to ARCHITECTURE.md documenting Rein's relationship to GSD, alongside existing comparisons to Ralph and Gas Town.

## Rationale

GSD is the most popular Claude Code "framework" (25.5K stars). Users discovering rein will ask "how does this relate to GSD?" The answer is clear: they operate at different layers and are complementary, not competitive.

## Proposed Content

Add to ARCHITECTURE.md under a "Ecosystem Positioning" or "Related Systems" section:

### Layer Model

```
┌─────────────────────────────────────────────┐
│  Human Workflow Layer                        │
│  GSD: spec → plan → execute → verify        │
│  (prompt engineering, in-process)            │
├─────────────────────────────────────────────┤
│  Agent Execution Layer                       │
│  Rein: dispatch → monitor → intervene     │
│  → evaluate → report                        │
│  (external process, subprocess monitoring)   │
├─────────────────────────────────────────────┤
│  Agent Runtime Layer                         │
│  Claude Code / Codex CLI / Gemini CLI       │
└─────────────────────────────────────────────┘
```

Key distinctions:
- **GSD** structures the human-to-agent conversation (what to build, in what order)
- **Rein** monitors the agent execution itself (token pressure, quality, cost)
- A GSD user could run tasks through rein for execution monitoring
- Rein does not need GSD; GSD does not need rein; both are enhanced by the other

## Target Document

ARCHITECTURE.md — add section after "Future: Multi-Agent"

## Verification

- [ ] Layer model accurately represents both systems
- [ ] No claims about GSD that require verification
- [ ] Cross-references GSD deep dive for details
