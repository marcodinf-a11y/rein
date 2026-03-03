# Agentic Harness — Report Format

Every `harness run` invocation produces a single JSON report capturing agent results and evaluation scores.

## Report JSON Schema

Reports are saved to `results/{task_id}_{YYYYMMDD_HHMMSS}.json`. One report per invocation — the `results[]` and `evaluations[]` arrays hold one entry per agent. When running against multiple agents, a single report file contains all agent results. The timestamp in the filename differentiates runs.

```json
{
    "task": {
        "id": "fizzbuzz-001",
        "name": "FizzBuzz implementation",
        "prompt": "...",
        "token_budget": 70000
    },
    "results": [
        {
            "agent_name": "claude-code",
            "task_id": "fizzbuzz-001",
            "exit_code": 0,
            "normalized_tokens": {
                "input_tokens": 12500,
                "output_tokens": 850,
                "cache_read_tokens": 9000,
                "cache_write_tokens": 3500,
                "total_tokens": 13350
            },
            "budget_status": "within",
            "budget_analysis": {
                "total_tokens": 13350,
                "budget": 70000,
                "utilization_pct": 19.1,
                "status": "within",
                "remaining": 56650,
                "cache_efficiency": {
                    "cache_read_tokens": 9000,
                    "cache_write_tokens": 3500
                }
            },
            "duration_seconds": 8.2,
            "cost_usd": 0.0399,
            "result_text": "...",
            "diff": "diff --git a/fizzbuzz.py...",
            "artifacts": {
                "fizzbuzz.py": "def main():\n..."
            },
            "timestamp": "2026-03-02T22:15:00"
        }
    ],
    "evaluations": [
        {
            "agent_name": "claude-code",
            "task_id": "fizzbuzz-001",
            "validation_passed": true,
            "validation_output": "1\n2\nFizz\n4\nBuzz\n...",
            "score": 1.0,
            "notes": ""
        }
    ],
    "timestamp": "2026-03-02T22:15:15"
}
```

## Key Fields

| Field | Description |
|---|---|
| `normalized_tokens` | Unified token usage, comparable across agents. |
| `budget_status` | Quick health check: `within`, `warning`, or `exceeded`. |
| `budget_analysis` | Detailed breakdown with utilization percentage, remaining budget, and cache efficiency. See [TOKENS.md](TOKENS.md) for full budget analysis details. |
| `cost_usd` | Dollar cost (available from Claude Code; `null` for other agents). |
| `diff` | Git diff of what the agent changed in the sandbox. |
| `artifacts` | Map of filename to content for files the agent created or modified. |
| `validation_passed` / `score` | Evaluation results from running validation commands. |

## Evaluation Scoring

Scoring is binary based on exit codes from [validation commands](TASKS.md):

- All commands exit 0 → `score = 1.0` (pass)
- Any command exits non-zero → `score = 0.0` (fail)

Stdout/stderr from validation commands is captured in `validation_output` for debugging.

## Output Directory

Reports are written to the `results/` directory. The filename pattern is:

```
results/{task_id}_{YYYYMMDD_HHMMSS}.json
```

The timestamp differentiates multiple runs of the same task.

## Budget Analysis in Reports

The `budget_analysis` block in each result entry provides a self-contained summary of token budget utilization, including a `cache_efficiency` sub-object that breaks out cache read and write tokens. For the full specification of budget tiers, normalized token accounting, and the warning/exceeded thresholds, see [TOKENS.md](TOKENS.md).
