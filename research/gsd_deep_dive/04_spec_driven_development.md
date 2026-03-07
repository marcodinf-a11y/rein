# GSD Deep Dive: Spec-Driven Development & Multi-Runtime

**March 2026**

This document analyzes GSD's spec-driven development workflow, XML task structure, `.spec` file patterns, and the installer's multi-runtime transformation layer.

---

## 1. Spec-Driven Development Philosophy

GSD's core thesis is that "vibecoding" (vague descriptions → AI generates code) produces inconsistent results because the LLM doesn't have enough context. GSD fixes this by extracting specifications before building:

From the README:
> "It's the context engineering layer that makes Claude Code reliable. Describe your idea, let the system extract everything it needs to know, and let Claude Code get to work."

The workflow enforces specification before execution:
1. `/gsd:new-project` — Extract vision, constraints, requirements, roadmap
2. `/gsd:discuss-phase` — Capture user preferences and design decisions
3. `/gsd:plan-phase` — Research domain, create verified atomic plans
4. `/gsd:execute-phase` — Execute plans with fresh context per plan
5. `/gsd:verify-work` — Verify against goals, not just test results

Each step produces artifacts that feed the next. The executor never works from vague descriptions — it works from XML-structured plans with specific files, actions, verification steps, and done criteria.

**Source:** README.md, ccforeveryone.com/gsd

---

## 2. The XML Task Structure

GSD plans use structured XML within markdown. This is GSD's most distinctive contribution:

```xml
<task type="auto">
  <name>Create login endpoint</name>
  <files>src/app/api/auth/login/route.ts</files>
  <action>
    Use jose for JWT (not jsonwebtoken - CommonJS issues).
    Validate credentials against users table.
    Return httpOnly cookie on success.
  </action>
  <verify>curl -X POST localhost:3000/api/auth/login returns 200 + Set-Cookie</verify>
  <done>Valid credentials return cookie, invalid return 401</done>
</task>
```

### Task Types

| Type | Frequency | Behavior |
|------|-----------|----------|
| `type="auto"` | ~90% | Execute autonomously, commit, continue |
| `type="checkpoint:human-verify"` | ~9% | Stop, present for user verification |
| `type="checkpoint:decision"` | ~0.9% | Stop, present decision options |
| `type="checkpoint:human-action"` | ~0.1% | Stop, user must perform manual action (2FA, etc.) |

### Task Attributes

Tasks can also have:
- `tdd="true"` — Execute in TDD cycle (RED → GREEN → REFACTOR)
- `depends_on` — Explicit dependency on another task within the plan

### Plan Frontmatter

Each PLAN.md has YAML frontmatter:

```yaml
---
phase: 3
plan: 2
type: standard  # standard | gap_closure
wave: 2
depends_on: [3-1]
autonomous: true
files_modified: [src/orders/api.ts, src/orders/types.ts]
requirements: [AUTH-01, AUTH-02]
---
```

The frontmatter enables:
- Wave grouping for parallel execution
- Dependency tracking across plans
- Requirement traceability (every requirement ID must appear in at least one plan)
- File conflict analysis (plans modifying the same files shouldn't be in the same wave)

**Source:** README.md, `agents/gsd-planner.md`, `agents/gsd-executor.md`

---

## 3. The Planner-Checker Verification Loop

Plans are not accepted on first draft. The workflow includes a verification loop:

1. **Planner** (gsd-planner) creates PLAN.md files
2. **Checker** (gsd-plan-checker) verifies:
   - Plans cover all phase requirement IDs
   - Tasks are specific and actionable (not vague)
   - Dependencies correctly identified
   - Waves assigned for parallel execution
   - Verification criteria exist for each task
   - Must-haves derived from phase goal
3. If issues found → **Revision** (planner updates plans based on checker feedback)
4. Max 3 iterations → user override

This is GSD's answer to the "garbage in, garbage out" problem. The checker is a separate subagent with its own fresh context, explicitly tasked with adversarial verification.

The checker also performs "Nyquist validation" — a cross-reference of plans against a VALIDATION.md strategy document created from research findings, ensuring test coverage aligns with the domain-specific validation architecture.

**Source:** `get-shit-done/workflows/plan-phase.md` (steps 8-12)

---

## 4. Deviation Rules

The executor has explicit rules for handling work discovered during execution:

| Rule | Trigger | Action | Permission |
|------|---------|--------|-----------|
| **Rule 1: Auto-fix bugs** | Code doesn't work (wrong queries, logic errors, type errors) | Fix inline, commit, track | Automatic |
| **Rule 2: Auto-add critical** | Missing error handling, validation, auth, rate limiting | Add, commit, track | Automatic |
| **Rule 3: Auto-fix blocking** | Missing dependency, wrong types, broken imports | Fix, commit, track | Automatic |
| **Rule 4: Ask about architecture** | New DB table, schema change, framework switch | STOP, checkpoint | User decision |

Additionally:
- **Scope boundary:** Only auto-fix issues caused by the current task. Pre-existing issues logged to `deferred-items.md`.
- **Fix attempt limit:** 3 auto-fix attempts per task, then document and continue.
- **Analysis paralysis guard:** 5+ consecutive Read/Grep/Glob calls without Edit/Write/Bash → STOP.

**Source:** `agents/gsd-executor.md` (deviation_rules section)

---

## 5. Multi-Runtime Support

GSD supports four runtimes: Claude Code, OpenCode, Gemini CLI, and Codex. The installer transforms the same core content into runtime-specific formats.

### Installation Targets

| Runtime | Install Location | Command Format | Agent Format |
|---------|-----------------|----------------|-------------|
| Claude Code | `~/.claude/` | `/gsd:command` (slash commands in `commands/gsd/`) | Agents in `agents/` with YAML frontmatter |
| OpenCode | `~/.config/opencode/` | `/gsd-command` (custom prompts) | Same markdown agents |
| Gemini CLI | `~/.gemini/` | `/gsd:command` | Same, with `AfterTool` instead of `PostToolUse` |
| Codex | `~/.codex/` | `$gsd-command` (skills) | Skills format (`skills/gsd-*/SKILL.md`) |

### Transformation Layer

The installer (`bin/install.js`) performs several transformations during installation:

1. **Path rewriting:** Replaces `$HOME/.claude/` with the target runtime's config path
2. **Hook registration:** Registers statusline and context monitor hooks in the runtime's `settings.json` format
3. **Command format adaptation:** Creates runtime-specific command files (Claude uses `commands/gsd/`, Codex uses `skills/gsd-*/SKILL.md`)
4. **Agent sandbox configuration:** Codex agents get `sandbox` declarations (e.g., `workspace-write` for executor, `read-only` for checker)

The core workflow content (markdown files in `get-shit-done/workflows/`, `get-shit-done/templates/`, `get-shit-done/references/`) is shared across all runtimes. The transformation is structural (file layout, path references, hook format), not semantic.

**Practical limitation:** The context monitor hook (`gsd-context-monitor.js`) reads Claude Code's statusline API. For Gemini CLI, it uses `AfterTool` instead of `PostToolUse`, but the underlying metrics bridge (`/tmp/claude-ctx-{session_id}.json`) depends on Claude Code's statusline format. Gemini support for context monitoring is likely incomplete.

**Source:** `bin/install.js`, README.md (installation section)

---

## 6. Model Profiles

GSD assigns different Claude models to different agent roles:

| Profile | Planning | Execution | Verification |
|---------|----------|-----------|--------------|
| `quality` | Opus | Opus | Sonnet |
| `balanced` | Opus | Sonnet | Sonnet |
| `budget` | Sonnet | Sonnet | Haiku |

This is a cost optimization strategy: use the strongest model for planning (where mistakes are most expensive) and cheaper models for execution (where plans provide sufficient guidance) and verification (where the scope is narrow).

The model profile is stored in `.planning/config.json` and resolved by the gsd-tools.cjs CLI. Each workflow reads the appropriate model from the init JSON.

**Source:** README.md (Model Profiles section), `get-shit-done/references/model-profiles.md`

---

## 7. Comparison: GSD Specs vs Rein Task Definitions

| Dimension | GSD PLAN.md | Rein TaskDefinition |
|-----------|------------|----------------------|
| **Format** | Markdown with XML tasks + YAML frontmatter | JSON (YAML planned) |
| **Granularity** | Multiple tasks per plan (2-5 typical) | One prompt per task |
| **Created by** | LLM planner agent | Human operator |
| **Verification built-in** | `<verify>` and `<done>` per task | `validation_commands` per task |
| **Dependency tracking** | `depends_on` in frontmatter | Not yet supported |
| **Requirement traceability** | `requirements: [AUTH-01, AUTH-02]` | `tags` for filtering |
| **Model assignment** | Via model profile (per agent role) | `model` field per task |
| **Deviation handling** | 4 deviation rules + fix limits | No deviation protocol |
| **Size constraint** | "Fits in fresh context window" | `token_budget` (70K default) |

### Key Differences

**GSD's plans are LLM-generated, rein tasks are human-written.** This is the fundamental difference. GSD trusts the LLM to decompose work and create actionable plans (with checker verification). Rein trusts the human operator to define tasks. Both approaches have tradeoffs:
- GSD: Higher automation, but plans may be incorrect or miss context
- Rein: Lower automation, but tasks reflect human understanding

**GSD bundles verification with execution.** Each XML task has `<verify>` and `<done>` inline. The executor runs verification as part of execution and only commits if criteria are met. Rein separates execution and validation — validation commands run after the agent exits. GSD's approach catches issues earlier; Rein's approach ensures validation is independent of the agent's self-assessment.

---

## 8. Implications for the Rein

### Study: Requirement Traceability
GSD's `requirements` field in plan frontmatter, combined with the checker's verification that all requirement IDs appear in plans, is a lightweight traceability mechanism. If rein adds multi-task workflows, a similar pattern could track which tasks address which goals.

### Study: Deviation Rules
GSD's 4-rule deviation framework (auto-fix bugs, auto-add critical, auto-fix blocking, ask about architecture) is a pragmatic guide for agent autonomy. Rein could include deviation guidelines in task prompts as defense-in-depth.

### Skip: LLM-Generated Plans
Rein's task definitions are intentionally human-written. The operator's understanding of the work is part of Rein's value — it ensures tasks are well-scoped and correctly targeted. LLM-generated plans add a layer of uncertainty that Rein's evaluation mission doesn't need.

### Skip: XML Task Format
Rein's JSON/YAML task format serves its purpose. XML-within-markdown is a GSD convention that doesn't add value for Rein's use case.

---

## Sources

- glittercowboy/get-shit-done: README.md, `agents/gsd-planner.md`, `agents/gsd-executor.md`, `agents/gsd-plan-checker.md`
- `get-shit-done/workflows/plan-phase.md` — planner-checker loop
- `get-shit-done/workflows/execute-phase.md` — execution protocol
- `bin/install.js` — multi-runtime transformation
- `get-shit-done/references/model-profiles.md` — model assignment
- Rein docs: TASKS.md (task definition format, validation)
- ccforeveryone.com/gsd — spec-driven development overview
