# Agentic Harness — Session Management

How long-running development sessions are tracked, when context degrades, and when to start fresh.

## Why Sessions Matter

AI coding agents degrade as conversations grow. More context means more tokens, which means higher cost, slower responses, and worse reasoning. The harness treats token consumption as the primary health metric for session quality.

A "session" in the harness is a single task execution: one task dispatched to one agent in one sandbox. The harness does not maintain multi-turn conversations with agents — each run is independent by default. Session continuation (multi-turn) is supported per-agent but is not the default workflow.

## Session Lifecycle

```
1. LOAD        Load task definition from JSON
                │
2. SANDBOX     Create temp directory
                Write seed files
                Run setup commands
                │
3. INVOKE      Start agent CLI as subprocess
                cwd = sandbox directory
                │
4. MONITOR     Agent runs, produces output
                (Token monitoring via JSON parsing)
                │
5. CAPTURE     Parse JSON/JSONL output
                Normalize token usage
                Capture diff and artifacts
                │
6. VALIDATE    Run validation_commands in sandbox
                Score: all pass → 1.0, any fail → 0.0
                │
7. REPORT      Generate structured JSON report
                Save to results/{task_id}_{timestamp}.json
                │
8. CLEANUP     Remove sandbox directory
                (Artifacts already captured)
```

This lifecycle corresponds to the execution flow defined in [ARCHITECTURE.md](ARCHITECTURE.md).

## Token Budget

Token budget monitoring — the 70k default, what counts toward the budget, budget status thresholds, and when to start fresh — is documented in [TOKENS.md](TOKENS.md).

## Session Continuation (Per-Agent)

By default, each harness run is a fresh, isolated execution. But agents support session continuation for multi-turn workflows:

### Claude Code

```bash
# Get session ID from first run
session_id=$(claude -p "query" --output-format json | jq -r '.session_id')

# Continue in that session
claude -p "next query" --resume "$session_id"

# Or continue the most recent session
claude -p "next query" --continue
```

### Codex CLI

```bash
# Resume last session
codex exec resume --last "next task"

# Resume specific session
codex exec resume <SESSION_ID>
```

### Gemini CLI

No documented session resume mechanism. Each invocation is independent.

## Report Format

The structured JSON report format — schema, key fields, evaluation scoring, and output conventions — is documented in [REPORTS.md](REPORTS.md).

## Multi-Session Patterns

For tasks that need iteration:

1. **Break it down** — split a large task into sequential subtasks, each with its own budget
2. **Fresh context per subtask** — each subtask gets a clean sandbox and a fresh agent session
3. **Carry forward artifacts** — use `files` in the next task's JSON to seed with output from the previous task
4. **Monitor trends** — if token usage is increasing across subtasks, the problem decomposition may need rethinking
