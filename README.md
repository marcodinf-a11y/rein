# Rein

Context-pressure-aware orchestrator for AI coding agents. Monitors how agents consume their context windows in real-time, intervenes when quality is at risk, and ensures work is persisted before degradation sets in. Dispatches tasks, isolates execution, captures output, evaluates results.

## Supported Agents

- **Claude Code** (`claude` CLI) — Anthropic
- **Codex CLI** (`codex` CLI) — OpenAI
- **Gemini CLI** (`gemini` CLI) — Google

## Documentation

| Document | Content |
|---|---|
| [BRIEF.md](BRIEF.md) | Project brief — what, who, why, five pillars, scope and positioning |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System overview, five subsystems (incl. pressure monitor), agent adapter protocol, execution flow, CLI design |
| [AGENTS.md](AGENTS.md) | Per-agent integration — invocation, JSON/NDJSON output formats, streaming events, token mappings, signal behavior |
| [TASKS.md](TASKS.md) | Task definition spec — JSON format, schema, examples, validation patterns |
| [ROADMAP.md](ROADMAP.md) | Release planning — phased scope, implementation modules, open decisions |
| [PRD_SKETCH.md](PRD_SKETCH.md) | Product requirements — functional (FR-001–098), non-functional, error handling, acceptance criteria |
| [TOKENS.md](TOKENS.md) | Token normalization, context pressure model (`ContextPressure`, `ZoneConfig`), budget thresholds, cache semantics |
| [REPORTS.md](REPORTS.md) | Report JSON schema, evaluation scoring, output conventions |
| [SESSIONS.md](SESSIONS.md) | Session management — lifecycle, **context pressure monitoring protocol**, zone actions, wrap-up, continuation |

## Quick Start

```bash
# Run a task against Claude Code
rein run -t tasks/example_fizzbuzz.json -a claude

# Run all tasks in a directory
rein run -t tasks/ -a claude

# Dry run
rein run -t tasks/example_fizzbuzz.json -a claude --dry-run
```

## What Happens When Things Go Wrong

**Agent not found:** If the agent CLI isn't installed or not in `PATH`, Rein errors immediately before creating a sandbox. `rein list agents` shows which agents are available.

**Context pressure kill:** When token consumption crosses the yellow zone (60%), Rein waits for the agent's current turn, then kills the subprocess. Uncommitted work is committed, a run log is written, and the report sets `termination_reason=context_pressure`. In the red zone (>80%), the kill is immediate. See [SESSIONS.md](SESSIONS.md) for details.

**Validation failure:** If any `validation_commands` exit non-zero, the report scores 0.0 and includes stdout/stderr from each failing command. The sandbox and all artifacts are preserved in the report before cleanup.

## Context Pressure

Rein monitors **context pressure** — the ratio of consumed tokens to the model's context window — as its primary metric. Configurable zones (Green 0–60%, Yellow 60–80%, Red >80%) trigger graceful wrap-up or immediate termination. See [SESSIONS.md](SESSIONS.md) for the full monitoring protocol and [TOKENS.md](TOKENS.md) for data structures.

## Token Budget

Default: **70,000 tokens** (input + output). Complementary cost/resource metric alongside context pressure. Thinking/reasoning tokens excluded. See [TOKENS.md](TOKENS.md) for the full token normalization model, budget thresholds, and analysis details.
