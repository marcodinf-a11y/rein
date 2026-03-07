# Ralph Wiggum Loop: Core Mechanism

**Deep Dive Document 01 | March 2026**

This document analyzes the Ralph Wiggum Loop's architecture, execution model, and context management strategy. It builds on the context degradation research in [`research/02_context_degradation_research.md`](../02_context_degradation_research.md).

---

## 1. The Loop

The Ralph Wiggum Loop is a bash-level orchestration pattern that runs an AI coding agent repeatedly until a task is complete:

```bash
while :; do cat PROMPT.md | claude ; done
```

Each iteration:
1. The agent process starts with a fresh context window
2. The full spec (`PROMPT.md`) is loaded into context
3. The agent reads the codebase from disk (files modified by prior iterations)
4. The agent picks one task from the spec, implements it, validates it
5. The agent commits the work via git
6. The agent exits
7. The loop restarts the process

The critical property is **statelessness between iterations**. No conversation history, no accumulated tool outputs, no growing context. Every iteration is a clean slate with identical starting context (the spec) plus the evolving codebase on disk.

---

## 2. Two Variants

### 2.1 Bash Loop (Original)

```bash
while :; do cat PROMPT.md | claude ; done
```

The process terminates and restarts. Context is truly discarded. This is the variant Huntley recommends for long-running loops.

**Properties:**
- Zero context carryover between iterations
- Clean process isolation (new PID each iteration)
- No risk of context compaction artifacts
- Higher startup latency (agent bootstraps each time)
- No way to carry forward in-session learnings except via files

### 2.2 Stop Hook (Claude Code Plugin)

The Claude Code plugin installs a stop hook that intercepts the agent's exit attempt. When Claude tries to stop, the hook checks for a "completion promise" string. If absent, it re-injects the original prompt with exit code 2, forcing the session to continue.

**Properties:**
- Context accumulates within the session
- Lower per-iteration latency (no process restart)
- Risk of compaction events — when context fills up, Claude Code compacts the conversation, potentially losing specification details
- Huntley warns: "the original Ralph approach terminates and restarts cleanly between tasks, unlike exit-hook plugins that force the same session to continue until done, causing context overflow and lossy compaction"

The stop hook variant is architecturally opposed to the technique's core insight. If the value of Ralph is context rotation, then preventing context rotation (by keeping the session alive) undermines the technique. The plugin's popularity likely stems from convenience (no bash scripting, built into Claude Code) rather than technical merit.

---

## 3. The Prompt Architecture

Ralph prompts follow a consistent structure:

### 3.1 Study Phase
```
Before making changes search codebase (don't assume not implemented)
Review specifications, fix_plan, and existing source code
```

The agent must read before writing. This prevents duplicate implementations — a common failure mode where the agent reimplements something that already exists from a prior iteration.

### 3.2 Implementation Phase
```
Choose the most important item and implement it
DO NOT IMPLEMENT PLACEHOLDER OR SIMPLE IMPLEMENTATIONS
```

One task per iteration. This constraint manages context window usage and prevents compound failures. The all-caps instruction against placeholders addresses a real problem: agents default to minimal implementations that pass type checks but do nothing.

### 3.3 Validation Phase
```
Run tests for the implemented unit
```

The agent runs validation before committing. This provides the feedback signal for future iterations — test failures appear in the codebase state and are visible to the next iteration.

### 3.4 Documentation Phase
```
Update fix_plan with learnings
When you learn something new about how to run the compiler update @AGENT.md
```

The agent updates persistent files that serve as inter-iteration memory. `fix_plan.md` tracks what's done and what remains. `AGENT.md` captures operational knowledge (build commands, compiler quirks).

### 3.5 Commit Phase
```
Git operations with semantic versioning (0.0.x)
```

Git commits serve as checkpoints. Each iteration's work is atomically captured. If a future iteration breaks something, prior state is recoverable via `git reset --hard`.

---

## 4. Context Management Strategy

Ralph's context management is simple: **don't manage context; discard it.**

Each iteration "mallocs" the full specification into a fresh context window. This is intentionally wasteful — the same spec is loaded every time, consuming tokens that could otherwise hold additional context. Huntley calls this "deterministic allocation":

> "Ralph is intentionally inefficient regarding context window allocation by 'mallocing arrays' repeatedly, allocating the full specification with each iteration, which serves a critical purpose of minimizing the risk of compaction events and context rot."

This strategy has a clear connection to the context degradation research:

| Degradation Factor | Ralph's Response |
|-------------------|-----------------|
| Sheer input length degrades reasoning (13.9-85%) | Keep context small by doing one task per iteration |
| Architectural collapse at 32K-128K tokens | Never approach the threshold — restart before accumulation |
| MECW << advertised MCW | Irrelevant — each iteration operates well within MECW |
| Lost in the middle (position bias) | The spec is always at the start of context; no "middle" to get lost in |
| Observation masking reduces cost 50% | Ralph achieves 100% observation masking by discarding the entire conversation |

The tradeoff is clear: Ralph pays a higher per-iteration cost (re-loading the spec) in exchange for zero context degradation risk. Rein's approach is more nuanced — it monitors degradation risk and intervenes only when necessary, preserving in-session context for as long as it remains useful.

---

## 5. The Git-as-Memory Pattern

Git serves three functions in the Ralph Loop:

### 5.1 State Persistence
Files on disk carry forward the agent's work. Each iteration reads the current state of the codebase, which includes all modifications from prior iterations. The agent doesn't need conversation history because the work product *is* the history.

### 5.2 Rollback Mechanism
When an iteration produces broken code, recovery options include:
- `git reset --hard` to the last known-good commit
- Running a separate "repair" prompt targeting the specific failure
- Feeding error logs to a different model for remediation planning

### 5.3 Progress Signal
Git history provides an objective record of what changed and when. The `fix_plan.md` file, updated each iteration, provides a human-readable progress log. Together, these replace the conversation transcript as the canonical record of work.

### Limitations
Git-as-memory works for *structural* state (code, configs, tests) but not for *reasoning* state (why a particular approach was chosen, what alternatives were considered, what trade-offs were made). Each iteration starts with zero knowledge of prior reasoning. The `AGENT.md` file partially addresses this by persisting operational knowledge, but it cannot capture the full reasoning chain.

---

## 6. Subagent Usage

Huntley's implementation uses aggressive subagent parallelism:
- **Up to 500 parallel subagents** for codebase analysis (search, read)
- **Single-threaded** for validation (1 subagent for build/test)

This asymmetry reflects the task structure: reading is parallelizable, writing is serializable. Rein's current design does not support intra-task subagent dispatch, though the planned multi-agent parallel execution could enable similar patterns.

---

## 7. Stopping Conditions

Ralph loops terminate when:
1. No more TODO items in `fix_plan.md`
2. All tests pass
3. The agent emits the "completion promise" string
4. The iteration cap is reached (safety limit)

The completion promise is a specific string (e.g., "COMPLETE") that the agent must output to signal genuine completion. The stop hook checks for this string; if absent, the loop continues. This is a simple but effective mechanism for distinguishing "I'm stuck" (agent exits without promise) from "I'm done" (agent exits with promise).

Without the completion promise, the loop relies on the iteration cap — a blunt instrument that provides cost control but not quality assurance. A loop that hits its iteration cap may have completed the work, failed entirely, or oscillated between partial solutions.

---

## Sources

- Huntley, Geoffrey. "Ralph Wiggum as a software engineer." ghuntley.com/ralph/
- Huntley, Geoffrey. "Everything is a ralph loop." ghuntley.com/loop/
- Anthropic. "Claude Code Ralph Wiggum Plugin." github.com/anthropics/claude-code/plugins/ralph-wiggum
- "Ralph Wiggum Loop." beuke.org
- "From ReAct to Ralph Loop." Alibaba Cloud Community
- Rein research/02_context_degradation_research.md
