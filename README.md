# Agentic Harness

Development workflow tool for controlling AI coding agents during long-running software development sessions. Dispatches tasks, isolates execution, captures output, evaluates results. Token tracking detects context rot risk.

**MVP**: single-agent workflow. **Future**: multi-agent (parallelism and comparison).

## Supported Agents

- **Claude Code** (`claude` CLI) — Anthropic
- **Codex CLI** (`codex` CLI) — OpenAI
- **Gemini CLI** (`gemini` CLI) — Google

## Documentation

| Document | Content |
|---|---|
| [BRIEF.md](BRIEF.md) | Project brief — what, who, why, four pillars, MVP scope, multi-agent future |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System overview, subsystems, agent adapter protocol, execution flow, CLI design |
| [AGENTS.md](AGENTS.md) | Per-agent integration reference — invocation syntax, JSON output formats, token field mappings, quirks |
| [TASKS.md](TASKS.md) | Task definition spec — JSON (MVP) + YAML (future), schema, examples, validation patterns |
| [TOKENS.md](TOKENS.md) | Token normalization model, budget thresholds, budget analysis, cache semantics |
| [REPORTS.md](REPORTS.md) | Report JSON schema, evaluation scoring, output conventions |
| [SESSIONS.md](SESSIONS.md) | Session management — lifecycle, session continuation, multi-session patterns |

## Quick Start

```bash
# Run a task against Claude Code
harness run -t tasks/example_fizzbuzz.json -a claude

# Run against all available agents
harness run -t tasks/example_fizzbuzz.json

# Run all tasks in a directory
harness run -t tasks/ -a claude

# Dry run
harness run -t tasks/example_fizzbuzz.json --dry-run
```

## Token Budget

Default: **70,000 tokens** (input + output). Thinking/reasoning tokens excluded. See [TOKENS.md](TOKENS.md) for the full token normalization model, budget thresholds, and analysis details.
