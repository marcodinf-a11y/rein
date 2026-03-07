# Proposal: STATE.md Pattern for LEARNINGS.md

**Priority:** Now | **Effort:** Low | **Source:** GSD Deep Dive + Ralph Wiggum Deep Dive

---

## Summary

When implementing LEARNINGS.md (proposed in the Ralph Wiggum deep dive), apply GSD's STATE.md size constraint pattern: enforce a < 100-line limit with explicit instructions to keep it as a digest, not an archive.

## Rationale

GSD's STATE.md is constrained to < 100 lines by explicit template instructions:
> "Keep STATE.md under 100 lines. It's a DIGEST, not an archive."

Without this constraint, agents will append to LEARNINGS.md indefinitely. Over multiple sessions, the file grows until it itself becomes a context pressure source — the exact problem it was designed to solve.

GSD uses a CLI tool (gsd-tools.cjs) for atomic state updates, preventing LLM corruption of structured markdown. The rein should consider similar structured management.

## Proposed Design

### LEARNINGS.md Template

```markdown
# Task Learnings

## Build & Test
- [Operational discovery from execution]
- [Compiler quirk, API pattern, test approach]

## Architecture Notes
- [Key decision or constraint discovered]

## Known Issues
- [Issue encountered but deferred]

---
*Max 80 lines. Keep recent, remove resolved. Digest, not archive.*
```

### Constraints

1. **Size limit:** 80 lines (leaving room for headers and metadata within 100 total)
2. **Instruction in task prompt:** "If LEARNINGS.md exceeds 80 lines, remove the oldest or least relevant entries before adding new ones."
3. **Scope:** Operational knowledge only — build commands, API quirks, test patterns. Not project status, not decisions, not progress.
4. **Format:** Simple markdown lists. No structured headers that agents might corrupt. No YAML frontmatter.

### Implementation

1. Add `LEARNINGS.md` as an optional seed file in task definitions
2. After task completion, capture the LEARNINGS.md from the sandbox
3. When re-dispatching the same task (or a related task), seed with the captured LEARNINGS.md
4. Include size constraint instruction in the task prompt template

## Target Documents

- SESSIONS.md — add LEARNINGS.md to seed file documentation
- TASKS.md — add LEARNINGS.md to the files/seed file section

## Verification

- [ ] Size constraint is enforced in prompt template
- [ ] LEARNINGS.md is captured after execution
- [ ] LEARNINGS.md is seeded into subsequent runs for the same task
- [ ] Content is operational knowledge, not project status
