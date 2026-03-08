# Rein — Report Format

Every `rein run` invocation produces a single JSON report capturing agent results and evaluation scores.

## Report JSON Schema

Reports are saved to `results/{task_id}_{YYYYMMDD_HHMMSS}.json`. One report per invocation — the `results[]` and `evaluations[]` arrays hold one entry per task execution. When running a directory of tasks, a single report file contains all results. The timestamp in the filename differentiates runs.

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
            "model": "sonnet",
            "effort": null,
            "task_id": "fizzbuzz-001",
            "exit_code": 0,
            "termination_reason": "completed",
            "normalized_tokens": {
                "input_tokens": 12500,
                "output_tokens": 850,
                "cache_read_tokens": 9000,
                "cache_write_tokens": 3500,
                "reload_tokens": 12500,
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
                },
                "reload_cost": {
                    "reload_tokens": 12500,
                    "reload_pct": 93.6
                }
            },
            "duration_seconds": 8.2,
            "cost_usd": 0.0399,
            "result_text": "...",
            "baseline_sha": "abc1234",
            "diff_patch": "diff --git a/fizzbuzz.py...",
            "diff_stat": "1 file changed, 15 insertions(+)",
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
            "completion_promise": true,
            "completion_confidence": "confident",
            "completion_summary": "Implemented FizzBuzz with all edge cases. Tests pass.",
            "notes": ""
        }
    ],
    "timestamp": "2026-03-02T22:15:15"
}
```

## Key Fields

| Field | Description |
|---|---|
| `model` | Model used for this execution (e.g. `sonnet`, `flash`), or `null` if not specified. |
| `effort` | Reasoning effort level used (`low`, `medium`, `high`), or `null` if not specified. |
| `normalized_tokens` | Unified token usage, consistent regardless of agent. Includes `reload_tokens` — the first-turn input token count representing context reload overhead. |
| `budget_status` | Quick health check: `within`, `warning`, or `exceeded`. |
| `budget_analysis` | Detailed breakdown with utilization percentage, remaining budget, cache efficiency, and reload cost. See [TOKENS.md](TOKENS.md) for full budget analysis details. |
| `cost_usd` | Dollar cost (available from Claude Code; `null` for other agents). |
| `baseline_sha` | Commit SHA recorded at sandbox creation, before agent invocation. Diff baseline for all workspace types. |
| `diff_patch` | Full unified diff (`git diff <baseline_sha> HEAD`). Also written to `results/{task_id}_{timestamp}.patch`. |
| `diff_stat` | Summary (`git diff --stat <baseline_sha> HEAD`). Used in condensed failure narrative (FR-093c). |
| `artifacts` | Map of filename to content for files the agent created or modified. |
| `termination_reason` | Why the run ended: `completed` (normal), `timed_out` (wall-clock limit), `context_pressure` (zone kill), or `error` (adapter failure). |
| `parse_error` | Exception message when NDJSON/JSONL parsing fails. `null` on success. When set, `normalized_tokens` is null and `result_text` is empty. |
| `raw_output` | Raw stdout/stderr captured on parse failure. `null` on successful parse. |
| `validation_passed` / `score` | Evaluation results from running validation commands. `bool \| null` and `float \| null` respectively. `null` when `validation_commands` is empty (no validation ran — distinct from pass or fail). |
| `completion_promise` | `boolean` — whether the agent wrote the `.rein/complete` marker file. |
| `completion_confidence` | `"confident" \| "suspicious" \| "overconfident" \| "incomplete" \| "unverified" \| "unevaluated"` — cross-referenced outcome (see matrix below). |
| `completion_summary` | Contents of the marker file (agent's self-assessment), or `null` if no marker. Empty string if marker exists but is empty. |

## Evaluation Scoring

Scoring is binary based on exit codes from [validation commands](TASKS.md):

- All commands exit 0 → `score = 1.0` (pass)
- Any command exits non-zero → `score = 0.0` (fail)
- Empty `validation_commands` → `score = null`, `validation_passed = null` (no validation ran)

Stdout/stderr from validation commands is captured in `validation_output` for debugging. When no validation commands are configured, `validation_output` is `null`.

### Completion Confidence

The completion promise (`.rein/complete` marker file) is cross-referenced against validation results to classify confidence. See [ADR-002](docs/adr/ADR-002-completion-promise-signal.md).

| Promise Filed | Validation Passed | Outcome | Interpretation |
|--------------|-------------------|---------|---------------|
| Yes | Yes | **Confident** | Agent believes it's done, and it is. |
| No | Yes | **Suspicious** | Validation passes but agent didn't signal completion. May indicate weak validation or agent ran out of context. |
| Yes | No | **Overconfident** | Agent claims done but validation fails. Possible hallucination or reward hacking. |
| No | No | **Incomplete** | Agent didn't finish and validation confirms it. Expected for context-pressure kills and timeouts. |
| Yes | null | **Unverified** | Agent claims done but no validation commands configured. Promise cannot be verified. |
| No | null | **Unevaluated** | No completion promise and no validation. No quality signal available. |

## Output Directory

Reports are written to the `results/` directory. The filename pattern is:

```
results/{task_id}_{YYYYMMDD_HHMMSS}.json
```

The timestamp differentiates multiple runs of the same task.

## Learnings Extraction

Each report includes a `learnings` section documenting what operational knowledge was extracted from the sandbox back to the project root (`.rein/LEARNINGS.md`). This runs after the final quality gate verdict — not after each retry round. See [ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md).

```json
{
    "learnings": {
        "extracted": true,
        "new_entries": [
            "- build: `cargo build --release` (debug builds fail timing tests)",
            "- test: `cargo test -- --test-threads=1` (DB tests not parallelizable)"
        ],
        "total_lines": 12,
        "warnings": []
    }
}
```

| Field | Description |
|-------|-------------|
| `extracted` | `boolean` — whether extraction ran (false if sandbox had no LEARNINGS.md) |
| `new_entries` | List of entries added to the project root file this session |
| `total_lines` | Total line count in `.rein/LEARNINGS.md` after merge |
| `warnings` | List of warning strings (e.g., "LEARNINGS.md exceeds 80 lines (currently 85), operator curation recommended") |

## Escalation Report

When the quality gate's final verdict is `"fail"` (all rounds exhausted), the report includes an `escalation_report` field — a self-contained failure narrative for the developer picking up the work. When the verdict is `"pass"` or `"warn"`, this field is `null`.

The escalation report is defined and documented in [QUALITY_GATE.md](QUALITY_GATE.md#escalation-report) and [ADR-012](docs/adr/ADR-012-structured-escalation-report.md).

```json
{
    "escalation_report": {
        "task_summary": "Refactor auth module to use JWT",
        "rounds_attempted": 4,
        "max_rounds": 4,
        "round_history": [
            {
                "round_number": 1,
                "approach_summary": "Round 1: tests failed (2/5), review requested changes.",
                "termination_reason": "completed",
                "failing_signals": [
                    {
                        "name": "tests",
                        "message": "2 of 5 tests failed (test_edge_case, test_empty_input)",
                        "output_excerpt": "AssertionError: parse_input(None) raised TypeError"
                    }
                ],
                "passing_signals": ["build", "lint", "context_pressure", "diff_size"]
            }
        ],
        "escalation_trigger": "max_rounds_exhausted",
        "completion_confidence": "overconfident",
        "learnings_snapshot": [
            "- test: `pytest tests/ --test-threads=1` (DB tests not parallelizable)"
        ],
        "diagnostic_summary": "Agent resolved 1 of 2 test failures across 2 rounds. Remaining: test_edge_case. Progress: improved.",
        "preserved_state": {
            "branch": "rein/refactor-auth-001-20260306",
            "last_commit": "a1b2c3d",
            "diff_stat": "5 files changed, 125 insertions(+), 30 deletions(-)"
        },
        "summary_agent_used": false
    }
}
```

| Field | Description |
|-------|-------------|
| `task_summary` | Task name — quick identifier |
| `rounds_attempted` / `max_rounds` | How many rounds ran vs. the configured limit |
| `round_history[]` | Per-round failure narrative with failing/passing signals |
| `round_history[].output_excerpt` | First 30 lines from failure marker, or last 30 lines. Full output in `rounds[].evaluation.validation_output` |
| `escalation_trigger` | Why escalation happened: `max_rounds_exhausted`, `context_pressure`, `timed_out`, or `error` |
| `completion_confidence` | Final round's completion promise cross-reference ([ADR-002](docs/adr/ADR-002-completion-promise-signal.md)): `overconfident` (agent claimed done, validation failed), `incomplete` (agent didn't claim done), or `unverified` (agent claimed done, no validation configured). `unevaluated` does not appear in escalation (no promise + no validation + non-fail verdict = nothing to escalate). |
| `learnings_snapshot` | Operational facts extracted from sandbox LEARNINGS.md ([ADR-011](docs/adr/ADR-011-learnings-extraction-after-final-verdict.md)) — context for the human picking up the work |
| `diagnostic_summary` | Overall assessment with progress classification (`improved` / `stagnated` / `regressed`) |
| `preserved_state` | Branch, commit SHA, diff stat — where to find the agent's work. `branch` is `null` for tempdir workspaces |
| `summary_agent_used` | Whether opt-in LLM enrichment was used (configurable via `[escalation]` in `rein.toml`) |

---

## Budget Analysis in Reports

The `budget_analysis` block in each result entry provides a self-contained summary of token budget utilization, including a `cache_efficiency` sub-object that breaks out cache read and write tokens, and a `reload_cost` sub-object that shows how much of the total token spend went to context setup (cold-start overhead). For the full specification of budget tiers, normalized token accounting, context reload cost tracking, and the warning/exceeded thresholds, see [TOKENS.md](TOKENS.md).
