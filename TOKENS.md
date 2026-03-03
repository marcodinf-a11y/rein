# Agentic Harness — Token Model & Budget

Single source of truth for token normalization, budget thresholds, and budget analysis.

## NormalizedTokenUsage

```python
@dataclass(frozen=True)
class NormalizedTokenUsage:
    """Unified token metrics across all agents."""
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    total_tokens: int = 0  # Computed: input + output

    def __post_init__(self):
        object.__setattr__(self, "total_tokens", self.input_tokens + self.output_tokens)
```

The `frozen=True` dataclass requires `object.__setattr__` in `__post_init__` to set `total_tokens` after initialization. This guarantees `total_tokens` is always `input_tokens + output_tokens` — never stale, never zero.

## Normalization Rules

- `total_tokens` is always `input_tokens + output_tokens` — never pulled from agent-specific "total" fields.
- Thinking/reasoning tokens are excluded from the total (agent-internal, not controllable).
- Cache tokens are tracked but not added to the total.
- Budget checks use `total_tokens`.

## Field Mapping

Each agent reports token usage differently. The harness normalizes into the fields above. See [AGENTS.md](AGENTS.md) for full per-agent extraction details.

| Normalized Field | Claude Code | Codex CLI | Gemini CLI |
|---|---|---|---|
| `input_tokens` | `usage.input_tokens` | sum of `usage.input_tokens` across `turn.completed` events | sum of `tokens.prompt` across models |
| `output_tokens` | `usage.output_tokens` | sum of `usage.output_tokens` across `turn.completed` events | sum of `tokens.candidates` across models |
| `cache_read_tokens` | `usage.cache_read_input_tokens` | sum of `usage.cached_input_tokens` across `turn.completed` events | sum of `tokens.cached` across models |
| `cache_write_tokens` | `usage.cache_creation_input_tokens` | 0 (not reported) | 0 (not reported) |

`cache_write_tokens` is only available from Claude Code. This asymmetry is documented, not hidden.

## Cache Semantics

The relationship between `input_tokens` and cache tokens varies by agent and is not fully specified by any provider:

- **Claude Code**: `input_tokens` may be exclusive of `cache_read_input_tokens` (it appears as a very small number alongside large cache values).
- **Codex CLI**: `input_tokens` may be inclusive or exclusive of `cached_input_tokens`.
- **Gemini CLI**: `prompt` may be inclusive or exclusive of `cached`.

The exact semantics need to be verified per-provider. Budget calculations depend on this interpretation — if `input_tokens` already includes cached tokens, double-counting is possible; if it excludes them, the budget underestimates actual context size. Until verified, the harness treats `input_tokens` and cache tokens as independent fields and does not adjust one based on the other.

## Budget Concept

Token tracking is central. The default budget is 70,000 tokens (`input_tokens + output_tokens`). This is not a hard limit — it is a health metric. It represents the point where context rot risk becomes significant for typical coding tasks.

What counts toward the budget:

- `input_tokens` + `output_tokens` = `total_tokens` (this is what the budget tracks)
- Thinking/reasoning tokens: **excluded** (agent-internal, not controllable)
- Cache read tokens: **tracked but not counted** toward budget
- Cache write tokens: **tracked but not counted** toward budget

## Budget Thresholds

| Status | Utilization | Total Tokens | Signal |
|---|---|---|---|
| **WITHIN** | < 80% | < 56,000 | Normal operation |
| **WARNING** | 80–100% | 56,000–70,000 | Context rot risk elevated |
| **EXCEEDED** | > 100% | > 70,000 | Context rot likely |

Recommended response:

- **WITHIN**: continue working.
- **WARNING**: finish current subtask, then start a new session.
- **EXCEEDED**: stop, start fresh, break the task into smaller pieces.

## BudgetStatus Enum

```python
class BudgetStatus(Enum):
    WITHIN = "within"       # Under 80% of budget
    WARNING = "warning"     # 80-100% of budget
    EXCEEDED = "exceeded"   # Over budget
```

## Budget Analysis

```python
def analyze_budget(usage, budget=70_000):
    return {
        "total_tokens": usage.total_tokens,
        "budget": budget,
        "utilization_pct": round(usage.total_tokens / budget * 100, 1),
        "status": usage.budget_status(budget).value,
        "remaining": max(0, budget - usage.total_tokens),
        "cache_efficiency": {
            "cache_read_tokens": usage.cache_read_tokens,
            "cache_write_tokens": usage.cache_write_tokens,
        },
    }
```

`cache_efficiency` is always included in the output. It provides visibility into how much of the input was served from cache versus freshly processed, which is useful for diagnosing cost and latency even though cache tokens do not count toward the budget.

Example output:

```json
{
    "total_tokens": 13350,
    "budget": 70000,
    "utilization_pct": 19.1,
    "status": "within",
    "remaining": 56650,
    "cache_efficiency": {
        "cache_read_tokens": 9000,
        "cache_write_tokens": 3500
    }
}
```

## Budget Override

**Choosing a budget:**

| Task complexity | Recommended budget |
|---|---|
| Simple (single file, clear spec) | 30,000–50,000 tokens |
| Medium (multi-file, some ambiguity) | 70,000 tokens (default) |
| Complex (multi-file, testing, iteration) | 70,000–100,000 tokens |

If a task consistently exceeds its budget, break it into smaller tasks rather than raising the budget.

**Override per-task:**

```json
{ "token_budget": 50000 }
```

**Override per-run:**

```bash
harness run -t tasks/my_task.json --budget 50000
```
