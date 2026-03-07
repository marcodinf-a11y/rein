# Missing for PRD

Gap analysis: what exists vs. what a Product Requirements Document needs before implementation.

## Already Covered

| PRD Section | Source | Coverage |
|---|---|---|
| Product vision & problem statement | BRIEF.md | Solid |
| Technical architecture | ARCHITECTURE.md | Comprehensive |
| Data models / schemas | TASKS.md, TOKENS.md, REPORTS.md | Detailed |
| Agent integration specs | AGENTS.md | Thorough |
| Known gaps & risks | INCONSISTENCIES.md | 19 issues cataloged |

## Missing

### 1. User Stories / Use Cases

No document describes what the user does end-to-end. Needed:

- Happy-path workflows (run task, view report, compare agents)
- Sad-path workflows (agent fails, timeout, invalid task file)
- Example user stories:
  - "As a developer, I want to run a coding task against Claude Code and see if it passes validation"
  - "As a developer, I want to compare token usage across agents for the same task"
  - "As a developer, I want to know when an agent is consuming too many tokens before context rot sets in"

### 2. Consolidated Functional Requirements

Requirements are scattered across 7 files. No single numbered list of trackable requirements (FR-001, FR-002, ...) that map to implementation. A PRD needs one place to answer "what must the system do?" without cross-referencing multiple specs.

### 3. Non-Functional Requirements

Completely absent:

- **Performance** — max acceptable harness overhead (excluding agent execution time)
- **Reliability** — behavior on agent crash, network failure, partial output
- **Compatibility** — Python version floor, OS support (Linux, macOS, Windows?)
- **Installability** — distribution method (pip, pipx, uv, git clone?)
- **Testability** — minimum test coverage expectations

### 4. Error Handling / Edge Cases

INCONSISTENCIES.md flags this gap but no spec exists. Needed:

- Agent timeout behavior (SIGTERM → SIGKILL sequence? grace period?)
- JSON/JSONL parse failure fallback (partial output? raw stderr?)
- Agent not installed or not authenticated
- Disk full during sandbox creation
- Task file missing required fields or malformed
- Validation command hangs indefinitely

### 5. Acceptance Criteria

No definition of "done" for MVP. Needed:

- What demo scenario proves it works end-to-end?
- What tests must pass before MVP is shippable?
- Minimum number of example tasks that must succeed?

### 6. Prioritized Scope / Phased Roadmap

MVP vs. future is mentioned casually across docs but there is no ordered backlog:

- **Phase 1 (MVP):** exact feature set that ships first
- **Phase 2:** what comes next? (multi-agent, YAML, brownfield?)
- **Phase 3:** longer-term (reporting dashboard, CI integration, plugin system?)

### 7. Assumptions & Constraints

Not explicitly listed anywhere. Examples that need documenting:

- Agents are already installed and authenticated by the user
- Git is available on the system
- Target Python version (3.12+? 3.11+?)
- Linux/macOS only, or Windows too?
- Internet connectivity required during execution?
- No root/admin privileges needed?

### 8. Success Metrics

No measurable goals defined. Examples:

- Harness overhead < N seconds per task
- Token normalization accuracy within N% of raw agent output
- MVP supports at least N task types end-to-end
- Report output matches schema with zero validation errors

### 9. Decisions Required Before Implementation

INCONSISTENCIES.md raises critical and high issues that need explicit decisions, not just identification:

| Issue | Decision Needed |
|---|---|
| Greenfield-only vs. brownfield | Is brownfield in MVP scope or deferred? |
| Cache token semantics | Are cached tokens included in `input_tokens` per agent? |
| Codex bare sandbox failure | Always pass `--skip-git-repo-check`, or require `git init` in setup? |
| "Not a benchmarking tool" vs. comparison features | Clarify positioning — comparison is a capability, not the purpose? |
| Context window monitoring | Ship in MVP or move to future? |
| Codex/Gemini turn limits | Enforce max turns, or leave to agent defaults? |

## Recommendation

Create a single `PRD.md` that:

1. Pulls vision and scope from BRIEF.md
2. Consolidates functional requirements from the 7 spec files into a numbered list
3. Adds the 9 missing sections above
4. Resolves the critical open decisions with explicit choices
