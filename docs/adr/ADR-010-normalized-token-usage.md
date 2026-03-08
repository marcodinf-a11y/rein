# ADR-010: Normalized Token Usage Across Agents

## Status

Accepted

## Context

The three supported agents report token usage in incompatible schemas:

- **Claude:** `input_tokens` (uncached only), `cache_read_input_tokens`, `cache_creation_input_tokens`, `output_tokens`. Input is exclusive of cache — all three partitions must be summed for total input.
- **Codex:** `input_tokens`, `output_tokens`, `cached_input_tokens`. Reported as per-turn deltas, not cumulative. No cache write field.
- **Gemini:** `prompt`, `candidates`, `cached`, `thoughts`, `total`. Different naming. `total` includes `thoughts`. May report across multiple models.

Rein's context pressure monitor and budget analysis need a single consistent token representation to compute pressure ratios and compare across agents. Passing raw agent-specific fields through would require every downstream consumer to handle three schemas.

## Decision

A unified `NormalizedTokenUsage` frozen dataclass with four fields:

| Field | Meaning |
|-------|---------|
| `input_tokens` | Total input tokens (all partitions summed) |
| `output_tokens` | Total output tokens |
| `cache_read_tokens` | Tokens served from cache |
| `cache_write_tokens` | Tokens written to cache (0 when agent doesn't report) |

`total_tokens` is computed in `__post_init__` as `input_tokens + output_tokens` — never from agent-reported "total" fields (Gemini's `total` includes `thoughts`, which would double-count).

Each adapter implements its own extraction logic:
- Claude: sums `input_tokens + cache_read_input_tokens + cache_creation_input_tokens`
- Codex: accumulates per-turn deltas across all `turn.completed` events
- Gemini: sums across all models in `stats.models`, maps `prompt` → `input_tokens`, `candidates` → `output_tokens`

## Consequences

**Positive:**

- Context pressure computation (`total_tokens / context_window`) is consistent regardless of agent. One formula, one code path.
- Budget analysis, reports, and trend logging all consume the same dataclass. No agent-specific branching downstream.
- Cache efficiency (`cache_read_tokens / input_tokens`) is comparable across agents.
- Adding a new agent requires only writing the extraction function — the rest of the system consumes `NormalizedTokenUsage` unchanged.

**Negative:**

- Normalization is lossy. Agent-specific fields that don't map to the four canonical fields are dropped (e.g., Gemini's `thoughts`, `tool` tokens). Raw output is preserved separately for debugging.
- Cache semantics differ between agents (Claude's exclusive input vs. Codex's inclusive input). The normalization makes them look the same, but the underlying accounting differs. This is documented in TOKENS.md but could mislead operators comparing cache efficiency across agents.
- `cache_write_tokens` is always 0 for Codex and Gemini (they don't report it). The field exists for Claude's benefit but carries no information for other agents.
