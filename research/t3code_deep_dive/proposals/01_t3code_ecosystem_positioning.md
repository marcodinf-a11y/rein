# Proposal: T3Code Ecosystem Positioning

**Target:** ARCHITECTURE.md (new subsection under "Future: Multi-Agent" or as a standalone "Landscape Context" section)
**Priority:** Now
**Effort:** Low

---

## Proposed Section: "T3Code Relationship"

The following section is proposed for addition to ARCHITECTURE.md.

---

### T3Code Relationship

T3Code (github.com/pingdotgg/t3code) is a TypeScript GUI-first orchestrator for coding agents (~2.5k stars, v0.0.4, actively maintained). It wraps agent CLIs (Codex, Claude Code) behind a provider adapter layer and exposes them through a React 19 web app and Electron desktop client. Its architecture uses Effect-based dependency injection, SQLite event sourcing, and git worktree isolation per session.

Rein and T3Code are complementary, not competitive:

| Dimension | T3Code | Rein |
|-----------|--------|------|
| **Primary interface** | Web/Desktop GUI | CLI |
| **Relationship to agent** | Wraps agent, provides UI and persistence | Monitors agent externally, enforces context budget |
| **Context management** | None (delegates to agent) | Real-time pressure monitoring, zone-based kill |
| **Quality evaluation** | None (user reviews in UI) | Structured binary evaluation (validation commands) |
| **Workspace isolation** | Git worktrees only | Tempdir, worktree, or copy |
| **Event system** | 47+ runtime + 20 orchestration events, SQLite event store | Structured JSON reports per execution |
| **Multi-provider** | Registry-based adapter switching (Codex, Claude Code) | `AgentAdapter` protocol (Claude, Codex, Gemini) |

**Key distinction:** T3Code enhances the agent interaction experience (UI, session replay, persistence). Rein enhances agent execution quality (monitoring, evaluation, intervention). A user could run agents through Rein for context management while using T3Code's UI for session visualization — these are different layers of the stack.

Rein is not a GUI and does not aspire to be one. Rein is infrastructure — a monitoring and enforcement layer that sits beneath any interface.

---

## Rationale

Rein docs reference Ralph, ralph-orchestrator, Gas Town, and GSD but not T3Code, which occupies a distinct niche as the most prominent GUI-first multi-provider agent orchestrator. Documenting the relationship clarifies Rein's positioning: infrastructure (monitoring + enforcement), not interface (GUI + persistence).

The comparison also clarifies what Rein should not attempt to build: web UIs, Electron apps, session replay, or event sourcing infrastructure. These are T3Code's domain.

## Source References

- [T3Code Deep Dive: Synthesis](../00_synthesis.md) — §7 (complementary positioning)
- [T3Code Deep Dive: Critical Analysis](../07_critical_analysis.md) — §3 (Rein ahead), §4 (T3Code ahead)
- [T3Code Deep Dive: Core Architecture](../01_core_architecture.md) — monorepo layout and tech stack
- ARCHITECTURE.md — current system overview, no T3Code reference
