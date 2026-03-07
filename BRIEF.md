# Rein — Project Brief

## What It Is

Rein is a **context-pressure-aware orchestrator** for AI coding agents. It monitors how agents consume their context windows during real development work, intervenes when quality is at risk, and ensures work is persisted before degradation sets in.

It sits above agents as a controller layer — dispatching tasks, isolating execution, monitoring context pressure in real-time, and managing graceful wrap-up when pressure thresholds are crossed.

Its purpose is context-pressure-aware orchestration during real development work. Structured, normalized reporting makes cross-agent comparison a natural capability — but comparison is a byproduct, not the goal.

## Who It's For

Developers who use AI coding agents (Claude Code, Codex CLI, Gemini CLI) for real software development and need a structured way to manage sessions, prevent context rot, and maintain quality across tasks.

## The Problem

AI coding agents are powerful but unmanaged. During a long session:

- **Context rot** — as the conversation grows, the agent's reasoning degrades. Research shows degradation is continuous and accelerating — it does not cliff at some safe threshold, and newer models with larger context windows exhibit the same patterns ([Chroma Research: Context Rot](https://research.trychroma.com/context-rot)). Token consumption relative to the model's context window is the leading indicator.
- **No pressure monitoring** — agents have no built-in mechanism to recognize they are degrading. By the time quality drops are visible to the user, significant context window has been consumed with diminishing returns.
- **No isolation** — agents operate directly in the working tree, making it hard to evaluate or roll back.
- **No structured evaluation** — success is eyeballed, not measured.
- **No cost visibility** — token usage and spend are buried in agent-specific formats.

Rein solves this by treating **context pressure** as the primary operational metric. You break work into discrete tasks sized to fit within safe context bounds, dispatch each to an agent in a sandbox, monitor context pressure in real-time via token streams, and intervene when pressure crosses configurable thresholds — ensuring work is committed and progress is logged before quality degrades.

## The Five Pillars

| Pillar | What It Does |
|---|---|
| **Task Dispatch** | Feed a structured task definition (prompt, seed files, setup commands) to an agent |
| **Execution Isolation** | Run the agent in its own sandbox (temp directory, worktree) so changes are contained |
| **Context Pressure Monitoring** | Track token consumption against the model's context window in real-time; classify into green/yellow/red zones; trigger wrap-up or termination when thresholds are crossed |
| **Output Capture** | Collect everything — artifacts, diffs, exit codes, raw JSON, token usage |
| **Evaluation** | Run validation commands, normalize token usage, score results, generate reports |

## Context Pressure as Core Metric

Context pressure — the ratio of consumed tokens to the model's context window — is Rein's primary operational metric. It drives all intervention decisions.

Rein monitors token consumption in real-time (via agent NDJSON/JSONL streams where available) and classifies pressure into configurable zones:

| Zone | Default | Rein Action |
|---|---|---|
| **Green** | 0–60% of context window | Continue execution |
| **Yellow** | 60–80% of context window | Graceful stop: wait for current turn, kill subprocess, Rein commits and logs |
| **Red** | >80% of context window | Immediate kill: Rein wraps up with partial data |

Zone thresholds are configurable globally and per-model — when models have large context windows but quality degrades early, shrink the effective ceiling. See [TOKENS.md](TOKENS.md) for data structures and [SESSIONS.md](SESSIONS.md) for the full monitoring protocol.

### Token Budget (Complementary)

The token budget (default 70,000 tokens) is a complementary cost/resource constraint, separate from context pressure. Context pressure tracks quality risk; the token budget tracks spend. Both are monitored. See [TOKENS.md](TOKENS.md).

## Scope

### Current: Single-Agent Workflow

The first version supports dispatching one task to one agent at a time. The flow is:

1. Load task from JSON
2. Create sandbox, seed files, run setup
3. Invoke agent CLI in sandbox
4. Parse output, normalize tokens
5. Run validation commands
6. Generate structured report

### Planned: Multi-Agent & YAML Support

Multi-agent support is planned for **workflow composition** — using different agents and models for different roles. For example: plan with Claude/Opus at high effort, implement with Gemini/Flash at medium effort, review with Claude/Sonnet. Each task targets a single agent.

- **Parallelism** — dispatch different tasks to different agents concurrently

The architecture supports this from day one (agent adapter protocol, normalized token model, per-task agent/model/effort fields) but parallel dispatch is not yet implemented. See [ROADMAP.md](ROADMAP.md) for release planning.

YAML task loading is a planned future addition. YAML's multiline block scalars, comments, and token efficiency make it attractive for human-authored task files. The `TaskDefinition` dataclass is format-agnostic — adding YAML support requires only a loader that detects file extension.

## Why Not the Anthropic Agent SDK?

The Anthropic Agent SDK is for *building* agents powered by Claude. Rein is for *controlling* existing CLI agents.

| Dimension | Anthropic Agent SDK | Rein |
|---|---|---|
| Scope | Build agents powered by Claude | Orchestrate existing CLI agents |
| Model support | Claude only | Model-agnostic |
| Abstraction | Agent loop, tools, sessions | Task dispatch, output capture, scoring |
| Use case | Creating new agents | Controlling existing agents |

## Supported Agents

- **Claude Code** (`claude` CLI) — Anthropic
- **Codex CLI** (`codex` CLI) — OpenAI
- **Gemini CLI** (`gemini` CLI) — Google
