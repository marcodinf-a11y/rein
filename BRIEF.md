# Agentic Harness — Project Brief

## What It Is

The agentic harness is a **development workflow tool** for controlling AI coding agents during long-running software development sessions. It sits above agents as a controller layer — dispatching tasks, isolating execution, capturing output, and evaluating results.

This is not a benchmarking or comparison tool. It is used during actual development work.

## Who It's For

Developers who use AI coding agents (Claude Code, Codex CLI, Gemini CLI) for real software development and need a structured way to manage sessions, prevent context rot, and maintain quality across tasks.

## The Problem

AI coding agents are powerful but unmanaged. During a long session:

- **Context rot** — as the conversation grows, the agent's reasoning degrades. Token consumption is the leading indicator.
- **No isolation** — agents operate directly in the working tree, making it hard to evaluate or roll back.
- **No structured evaluation** — success is eyeballed, not measured.
- **No cost visibility** — token usage and spend are buried in agent-specific formats.

The harness solves this by giving you a controller layer. You break work into discrete tasks, dispatch each to an agent in a sandbox, monitor token health, and evaluate what came back.

## The Four Pillars

| Pillar | What It Does |
|---|---|
| **Task Dispatch** | Feed a structured task definition (prompt, seed files, setup commands) to an agent |
| **Execution Isolation** | Run the agent in its own sandbox (temp directory, worktree) so changes are contained |
| **Output Capture** | Collect everything — artifacts, diffs, exit codes, raw JSON, token usage |
| **Evaluation** | Run validation commands, normalize token usage, score results, generate reports |

## Token Budget as Health Metric

Token tracking is central. The default budget is **70,000 tokens** (input + output). This is not a hard limit — it's a health metric that signals context rot risk. Thinking/reasoning tokens are excluded from the budget. Cache tokens are tracked but not counted toward the total.

For budget thresholds, status tiers, and the full token normalization model, see [TOKENS.md](TOKENS.md).

## Scope

### MVP: Single-Agent Workflow

The first version supports dispatching one task to one agent at a time. The flow is:

1. Load task from JSON
2. Create sandbox, seed files, run setup
3. Invoke agent CLI in sandbox
4. Parse output, normalize tokens
5. Run validation commands
6. Generate structured report

### Future: Multi-Agent & YAML Support

Multi-agent support is planned for two use cases:

- **Comparison** — run the same task against multiple agents, compare results
- **Parallelism** — dispatch different tasks to different agents concurrently

The architecture supports this from day one (agent adapter protocol, normalized token model) but the MVP does not implement parallel dispatch.

YAML task loading is a planned future addition. YAML's multiline block scalars, comments, and token efficiency make it attractive for human-authored task files. The `TaskDefinition` dataclass is format-agnostic — adding YAML support requires only a loader that detects file extension.

## Why Not the Anthropic Agent SDK?

The Anthropic Agent SDK is for *building* agents powered by Claude. The agentic harness is for *controlling* existing CLI agents.

| Dimension | Anthropic Agent SDK | Agentic Harness |
|---|---|---|
| Scope | Build agents powered by Claude | Orchestrate existing CLI agents |
| Model support | Claude only | Model-agnostic |
| Abstraction | Agent loop, tools, sessions | Task dispatch, output capture, scoring |
| Use case | Creating new agents | Controlling existing agents |

## Supported Agents

- **Claude Code** (`claude` CLI) — Anthropic
- **Codex CLI** (`codex` CLI) — OpenAI
- **Gemini CLI** (`gemini` CLI) — Google
