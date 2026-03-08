# Rein — Agent Integration Reference

Per-agent invocation syntax, JSON output formats, token field mappings, and quirks.

---

## Claude Code (`claude` CLI)

### Invocation

```bash
echo "PROMPT" | claude -p --output-format json --dangerously-skip-permissions --max-turns 50
```

Run via `asyncio.create_subprocess_exec` with `cwd` set to the sandbox directory. The prompt is delivered via stdin (piped to the process) rather than as a CLI argument — this avoids `ARG_MAX` limits on Linux for long prompts and is the standard delivery method for all agents.

### Key CLI Flags

| Flag | Purpose |
|---|---|
| `-p` / `--print` | Non-interactive (headless) mode |
| `--output-format json` | Structured JSON output with result, session ID, metadata |
| `--output-format stream-json` | Newline-delimited JSON for real-time streaming |
| `--dangerously-skip-permissions` | Skip all permission prompts |
| `--allowedTools "Bash,Read,Edit"` | Auto-approve specific tools |
| `--max-budget-usd <amount>` | Stop execution at cost threshold |
| `--max-turns <n>` | Limit agentic iterations |
| `--continue` / `-c` | Continue most recent conversation |
| `--resume <session_id>` | Resume a specific session |
| `--model <model>` | Select model (e.g. `claude-opus-4-6`, `opus`) |
| `--append-system-prompt` | Add custom instructions |

### JSON Output Format

```json
{
    "type": "result",
    "subtype": "success",
    "is_error": false,
    "duration_ms": 3302,
    "duration_api_ms": 2918,
    "num_turns": 1,
    "result": "Hello! I'm Claude...",
    "session_id": "3892c079-721c-4e63-b06b-38a0be87809b",
    "total_cost_usd": 0.03988395,
    "usage": {
        "input_tokens": 3,
        "cache_creation_input_tokens": 9443,
        "cache_read_input_tokens": 13829,
        "output_tokens": 21,
        "server_tool_use": {
            "web_search_requests": 0,
            "web_fetch_requests": 0
        },
        "service_tier": "standard",
        "cache_creation": {
            "ephemeral_1h_input_tokens": 9443,
            "ephemeral_5m_input_tokens": 0
        }
    },
    "modelUsage": {
        "claude-sonnet-4-5-20250929": {
            "inputTokens": 3,
            "outputTokens": 21,
            "cacheReadInputTokens": 13829,
            "cacheCreationInputTokens": 9443,
            "costUSD": 0.03988395,
            "contextWindow": 200000,
            "maxOutputTokens": 64000
        }
    }
}
```

### Stream Output Format (for Context Pressure Monitoring)

With `--output-format stream-json`, Claude emits newline-delimited JSON (NDJSON). Key event types for mid-run monitoring:

```jsonl
{"type":"system","subtype":"init","session_id":"...","tools":[...]}
{"type":"assistant","message":{"id":"msg_...","usage":{"input_tokens":12500,"output_tokens":0}},"subtype":"message_start"}
{"type":"assistant","message":{"usage":{"input_tokens":12500,"output_tokens":347}},"subtype":"message_delta"}
{"type":"result","subtype":"success","session_id":"...","result":"...","usage":{...},"modelUsage":{...}}
```

- `message_start`: carries initial `input_tokens` for this API call
- `message_delta`: carries cumulative `output_tokens` within the current message
- `result`: final event with full aggregated usage (same schema as `--output-format json`)

Rein reads this stream line by line, updating cumulative token counts and computing context pressure after each `message_delta` event. The `contextWindow` field from `modelUsage` in the final `result` event provides the denominator — on first run, rein may not know the context window until this event arrives; for known models, the model metadata lookup provides it at invocation time.

**Caveat:** When extended thinking mode is enabled, `StreamEvent` emission is disabled. The stream produces only the final `result` event. Mid-run context pressure monitoring is not available in this mode.

### Token Field Mapping

```python
def _normalize_claude(data: dict) -> NormalizedTokenUsage:
    usage = data.get("usage", {})
    # Claude's input_tokens is EXCLUSIVE of cache — it reports only the
    # uncached tail.  Sum all three partitions to get total input tokens.
    raw_input = usage.get("input_tokens", 0)
    cache_read = usage.get("cache_read_input_tokens", 0)
    cache_write = usage.get("cache_creation_input_tokens", 0)
    return NormalizedTokenUsage(
        input_tokens=raw_input + cache_read + cache_write,
        output_tokens=usage.get("output_tokens", 0),
        cache_read_tokens=cache_read,
        cache_write_tokens=cache_write,
    )
```

### Session Resume

```bash
session_id=$(claude -p "query" --output-format json | jq -r '.session_id')
claude -p "next query" --resume "$session_id"
```

### Quirks and Limitations

- **Two naming conventions**: The top-level `usage` block uses **snake_case** (`input_tokens`), while `modelUsage` uses **camelCase** (`inputTokens`). Parse from `usage` — it is the aggregate across all models.
- **Cannot run inside another Claude Code session**: When the `CLAUDECODE` env var is set, `claude -p` fails. The adapter scrubs this variable before spawning the subprocess. See [ARCHITECTURE.md — Environment variable handling](ARCHITECTURE.md#2-execution-isolation-sandboxpy) for the cross-cutting env var policy.
- **Cost available**: `total_cost_usd` at top level, `costUSD` per model in `modelUsage`.
- **Effort control**: Set via `CLAUDE_CODE_EFFORT_LEVEL` env var on the subprocess (`low`, `medium`, `high`). No CLI flag. Supported on Opus 4.6 and Sonnet 4.6 only. The adapter sets this env var when the task or CLI specifies an `effort` value.

---

## Codex CLI (`codex` CLI)

### Invocation

```bash
echo "PROMPT" | codex exec --json --full-auto --skip-git-repo-check
```

### Key CLI Flags

| Flag | Purpose |
|---|---|
| `exec` (alias: `e`) | Non-interactive execution mode |
| `--json` / `--experimental-json` | JSON Lines output for machine consumption |
| `--ephemeral` | Don't persist session rollout files |
| `--full-auto` | Auto-approve edits (combines approvals + workspace-write sandbox) |
| `--sandbox <mode>` | `read-only` \| `workspace-write` \| `danger-full-access` |
| `-o, --output-last-message <path>` | Write final message to file |
| `--skip-git-repo-check` | Override Git repository requirement |
| `-m, --model` | Override model (e.g. `gpt-5-codex`) |
| `--yolo` | Remove all safeguards (alias for `--dangerously-bypass-approvals-and-sandbox`) |

### JSON Output Format (JSONL Stream)

With `--json` enabled, stdout is a JSON Lines stream. Key event types:

```jsonl
{"type":"thread.started","thread_id":"0199a213-81c0-7800-8aa1-bbab2a035a53"}
{"type":"turn.started"}
{"type":"item.completed","item":{"id":"item_3","type":"agent_message","text":"..."}}
{"type":"turn.completed","usage":{"input_tokens":24763,"cached_input_tokens":24448,"output_tokens":122}}
```

Event types: `thread.started`, `turn.started`, `turn.completed`, `turn.failed`, `item.started`, `item.completed`, `error`.

Token usage appears on `turn.completed` events as **per-turn deltas** (not cumulative). A single execution may have multiple turns — all must be summed for total usage.

**Mid-run monitoring:** Events arrive in real-time, not buffered. Rein reads this stream line by line, summing `input_tokens` and `output_tokens` from each `turn.completed` event to compute cumulative usage and context pressure. If the process is killed mid-turn, that turn's usage is not emitted. On `SIGINT`, Codex emits a `TurnAborted` event before exiting.

### Token Field Mapping

```python
def _parse_codex_jsonl(stdout: str) -> tuple[dict, NormalizedTokenUsage]:
    total_input = 0
    total_output = 0
    total_cached = 0
    last_message = ""
    errors: list[str] = []

    for line in stdout.strip().splitlines():
        event = json.loads(line)
        event_type = event.get("type")
        if event_type == "turn.completed":
            usage = event.get("usage", {})
            total_input += usage.get("input_tokens", 0)
            total_output += usage.get("output_tokens", 0)
            total_cached += usage.get("cached_input_tokens", 0)
        elif event_type == "item.completed":
            item = event.get("item", {})
            if item.get("type") == "agent_message":
                last_message = item.get("text", "")
        elif event_type == "turn.failed":
            errors.append(event.get("error", "turn failed"))
        elif event_type == "error":
            errors.append(event.get("message", "unknown error"))

    result: dict = {"last_message": last_message}
    if errors:
        result["errors"] = errors

    return (
        result,
        NormalizedTokenUsage(
            input_tokens=total_input,
            output_tokens=total_output,
            cache_read_tokens=total_cached,
            cache_write_tokens=0,  # Codex does not report cache writes
        ),
    )
```

### Session Resume

```bash
codex exec resume --last "next task"
codex exec resume <SESSION_ID>
```

### Quirks and Limitations

- **No `cache_creation` concept**: `cached_input_tokens` maps to `cache_read_tokens`; `cache_write_tokens` is always 0.
- **Requires a Git repository** by default. The adapter always passes `--skip-git-repo-check` because sandboxes may not contain a `.git` directory.
- **Runs in read-only sandbox** by default. `--full-auto` upgrades to `workspace-write`. Rein uses `--full-auto` rather than `--yolo` because `--full-auto` auto-approves edits while preserving Codex's network and process sandboxing — `--yolo` disables all safeguards including the sandbox.
- **Progress on stderr, results on stdout**: stderr gets streaming progress; stdout gets JSON events.
- **Authentication**: `CODEX_API_KEY` env var (only supported in `codex exec` mode).
- **Effort control**: Set via `--config 'model_reasoning_effort="<level>"'` (`minimal`, `low`, `medium`, `high`, `xhigh`). Requires Responses API wire protocol. Works with o3, o4-mini, gpt-5. Rein maps normalized `low`/`medium`/`high` values directly; Codex-specific `minimal` and `xhigh` are not available via the normalized `effort` field.

---

## Gemini CLI (`gemini` CLI)

### Invocation

```bash
echo "PROMPT" | gemini --output-format json --yolo
```

Prompt is delivered via stdin for all agents. See Claude invocation note for rationale.

### Key CLI Flags

| Flag | Purpose |
|---|---|
| `--prompt`, `-p` | Non-interactive (headless) mode |
| `--output-format` | Output format: `text` (default) or `json` |
| `--model`, `-m` | Select model (e.g. `gemini-2.5-flash`, `gemini-2.5-pro`) |
| `--yolo`, `-y` | Auto-approve all actions |
| `--approval-mode` | Set approval behavior (e.g. `auto_edit`) |
| `--debug`, `-d` | Enable debug mode |
| `--all-files`, `-a` | Include all files in context |
| `--include-directories` | Add specific directories to context |

### JSON Output Format

**Single object mode** (`--output-format json`):

```json
{
    "response": "string",
    "stats": {
        "models": {
            "gemini-2.5-pro": {
                "api": {
                    "totalRequests": 5,
                    "totalErrors": 0,
                    "totalLatencyMs": 12345
                },
                "tokens": {
                    "prompt": 24939,
                    "candidates": 20,
                    "total": 25113,
                    "cached": 21263,
                    "thoughts": 154,
                    "tool": 0
                }
            }
        },
        "tools": {
            "totalCalls": 3,
            "totalSuccess": 3,
            "totalFail": 0,
            "totalDurationMs": 5000,
            "totalDecisions": {
                "accept": 3,
                "reject": 0,
                "modify": 0,
                "auto_accept": 3
            }
        },
        "files": {
            "totalLinesAdded": 10,
            "totalLinesRemoved": 2
        }
    }
}
```

**Stream mode** (`--output-format stream-json`): Emits NDJSON with event types `message`, `tool_use`, `tool_result`, `error`, and `result`. Token counts appear **only in the final `result` event** (inside `StreamStats`), not in intermediate events. This means mid-run context pressure monitoring is not available via the stream alone. Each NDJSON line is independently parseable, which avoids the invalid JSON problem that can occur with single-object mode.

**Future path for mid-run tokens:** Gemini CLI supports OpenTelemetry export. The `gen_ai.client.token.usage` metric provides per-operation token counts. This would enable mid-run monitoring but requires an OTel collector or export parser — documented as a future enhancement.

### Token Fields

| Field | Meaning |
|---|---|
| `prompt` | Input tokens sent |
| `candidates` | Response tokens generated |
| `total` | Combined count (**includes** `thoughts`) |
| `cached` | Tokens reused from cache |
| `thoughts` | Internal reasoning/chain-of-thought tokens |
| `tool` | Tool-related tokens |

### Token Field Mapping

Output may contain multiple models — sum across all of them:

```python
def _normalize_gemini(data: dict) -> NormalizedTokenUsage:
    models = data.get("stats", {}).get("models", {})
    total_prompt = 0
    total_candidates = 0
    total_cached = 0

    for model_name, model_data in models.items():
        tokens = model_data.get("tokens", {})
        total_prompt += tokens.get("prompt", 0)
        total_candidates += tokens.get("candidates", 0)
        total_cached += tokens.get("cached", 0)

    return NormalizedTokenUsage(
        input_tokens=total_prompt,
        output_tokens=total_candidates,
        cache_read_tokens=total_cached,
        cache_write_tokens=0,  # Gemini does not report cache writes
    )
```

### Session Resume

No documented session resume mechanism.

### Quirks and Limitations

- **`prompt` / `candidates`** instead of `input` / `output` — different naming from the other two agents.
- **`total` includes `thoughts`**: Gemini's own `total` field includes internal reasoning tokens. Rein computes its own total as `prompt + candidates` for consistency.
- **JSON output can be unreliable**: GitHub issue #11184 reports `--output-format json` sometimes produces invalid JSON. The adapter catches `json.JSONDecodeError` and falls back: `normalized_tokens` is null, `result_text` is empty, raw stdout is preserved in `raw_output`, and the report sets `parse_error` with the exception message. See [ARCHITECTURE.md — Output Capture](ARCHITECTURE.md#4-output-capture-agentspy) for the full fallback protocol.
- **No cost reporting**: Gemini CLI does not report cost in its output. Maps to `null` in results.
- **Free tier limits**: 60 requests/min, 1,000 requests/day, 1M token context window on Gemini 2.5 Pro.
- **Effort control**: No CLI flag or env var. Requires a custom model alias in `settings.json` with a `thinkingConfig` block. Gemini 2.5 uses integer `thinkingBudget`; Gemini 3 uses `thinkingLevel` enum (`minimal`/`low`/`medium`/`high`). The adapter must manage settings dynamically — this is the most complex effort integration of the three agents.

---

## Cross-Agent Reference

| Capability | Claude Code | Codex CLI | Gemini CLI |
|---|---|---|---|
| Output format | JSON (`--output-format json`) or NDJSON stream (`--output-format stream-json`) | JSONL stream (`--json`) | JSON (`--output-format json`) or NDJSON stream (`--output-format stream-json`) |
| Mid-run token streaming | Yes (NDJSON `message_start`/`message_delta` events with usage). Disabled when extended thinking is active. | Yes (JSONL `turn.completed` events with per-turn usage deltas) | No (tokens only in final `result` event). OpenTelemetry `gen_ai.client.token.usage` is a future alternative. |
| Context window reported | Yes (`modelUsage.<model>.contextWindow`) | No (must use model metadata lookup) | No (must use model metadata lookup) |
| Token naming | `input_tokens` / `output_tokens` | `input_tokens` / `output_tokens` | `prompt` / `candidates` |
| Cache read | `cache_read_input_tokens` | `cached_input_tokens` | `cached` |
| Cache write | `cache_creation_input_tokens` | Not reported | Not reported |
| Cost reporting | `total_cost_usd` | Not reported | Not reported |
| Session resume | `--resume <id>`, `--continue` | `resume <id>`, `resume --last` | Not available |
| Auto-approve | `--dangerously-skip-permissions` | `--full-auto` | `--yolo` |
| Headless mode | `-p` | `exec` | `-p` |
| Git identity example | `claude-code/opus/high` | `codex-cli/o3/high` | `gemini-cli/pro` |
| Graceful stop signal | `SIGTERM` (exit 143, no final output) | `SIGINT` (emits `TurnAborted` in JSONL; second `SIGINT` = hard kill) | `SIGTERM` (clean exit code 0, no final result event) |
| Timeout behavior | [Subprocess Termination Procedure](ARCHITECTURE.md#subprocess-termination-procedure) (immediate): `SIGTERM` → 5s → `SIGKILL`. All complete NDJSON lines preserved. Exit code typically `-15` or `-9`. | Subprocess Termination Procedure (immediate): `SIGINT` → 5s → `SIGKILL`. All complete JSONL lines preserved. Exit code typically `130` or `-9`. | Subprocess Termination Procedure (immediate): `SIGTERM` → 5s → `SIGKILL`. All complete NDJSON lines preserved. Exit code typically `-15` or `-9`. |
