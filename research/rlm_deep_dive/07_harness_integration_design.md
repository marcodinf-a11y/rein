# RLM Rein Integration Design

**Deep Dive Document 07 | March 2026**

A concrete integration design for bringing RLM capabilities into rein. This document specifies interfaces, data flow, and trade-offs. It does not recommend a single path -- it presents options with their costs.

References: [ARCHITECTURE.md](../../ARCHITECTURE.md), [TOKENS.md](../../TOKENS.md), [SESSIONS.md](../../SESSIONS.md), [QUALITY_GATE.md](../../QUALITY_GATE.md), [01_theory_mechanism.md](01_theory_mechanism.md), [02_benchmarks_evaluation.md](02_benchmarks_evaluation.md).

---

## 1. RlmAdapter Design

The `RlmAdapter` must satisfy the existing `AgentAdapter` protocol:

```python
@runtime_checkable
class AgentAdapter(Protocol):
    @property
    def name(self) -> str: ...
    def is_available(self) -> bool: ...
    async def run(self, task: TaskDefinition, sandbox_path: Path) -> AgentResult: ...
```

### In-Process vs. Subprocess

Every existing adapter (Claude Code, Codex, Gemini) invokes an external CLI via `asyncio.create_subprocess_exec`. The RLM library (`rlms`) is a Python package, not a CLI. Two options:

**Option A: In-process (`import rlms`).** The adapter imports the `rlms` library directly, calls `completion()` within Rein's Python process, and collects results. The `run()` method is still `async` -- it wraps the synchronous `rlms` call in `asyncio.to_thread()` or uses the library's async API if available.

| Pro | Con |
|-----|-----|
| Direct access to `RLMLogger` events, sub-call metadata, REPL state | Couples rein to the `rlms` Python package as a dependency |
| Token usage available programmatically -- no stream parsing needed | Crashes in RLM code (REPL exceptions, OOM) can bring down rein process |
| Lower latency -- no subprocess overhead | Breaks Rein's design principle: "No ML/AI libraries -- rein only *invokes* agents" (ARCHITECTURE.md) |

**Option B: Subprocess wrapper.** Write a thin CLI script (`rlm_runner.py`) that accepts a task prompt on stdin or as an argument, invokes `rlms.completion()`, and emits NDJSON on stdout matching Rein's stream parsing expectations. The adapter invokes this script via `asyncio.create_subprocess_exec`, identical to existing adapters.

| Pro | Con |
|-----|-----|
| Process isolation -- RLM crashes do not affect rein | Requires designing a NDJSON output protocol for RLM events |
| Consistent with existing adapter architecture | Token metadata must be serialized/deserialized across the process boundary |
| No new dependencies in rein package | Additional maintenance surface (the wrapper script) |

**Assessment.** Option B is the safer choice for initial integration. It preserves process isolation and architectural consistency. Option A becomes attractive if rein later needs real-time control over individual sub-calls (e.g., killing a specific sub-call that exceeds a token threshold), but that level of control is not required for MVP integration.

### Task-to-RLM Mapping

The adapter maps `task.prompt` to the RLM `completion()` call. The key question is what becomes the RLM's "context" variable versus its "instruction."

```python
class RlmAdapter:
    @property
    def name(self) -> str:
        return "rlm"

    async def run(self, task: TaskDefinition, sandbox_path: Path) -> AgentResult:
        # 1. Build the RLM context from sandbox contents
        context = self._build_context(sandbox_path, task)

        # 2. The task prompt becomes the RLM instruction
        instruction = task.prompt

        # 3. Invoke RLM (subprocess or in-process)
        result = await self._invoke_rlm(instruction, context, sandbox_path)

        # 4. Apply RLM output to sandbox (write files, run commands)
        await self._apply_result(result, sandbox_path)

        return result
```

**Context construction.** For `workspace.type = "worktree"` or `"copy"`, the sandbox contains an existing codebase. The adapter reads the file tree and concatenates relevant files into the RLM's context variable. For `"tempdir"`, context comes only from `task.files`. The adapter should respect `.gitignore` and skip binary files when building context.

**Sandbox interaction.** The RLM's REPL environment and rein sandbox are separate concepts. The RLM REPL is where the model executes Python code to analyze context and delegate sub-calls. Rein sandbox is the filesystem where the final code artifacts live. The adapter must bridge these: after the RLM completes, it extracts code artifacts from the RLM's output and writes them to `sandbox_path`. The RLM does not directly modify the sandbox filesystem -- the adapter mediates.

### LLM Backend Configuration

The RLM framework requires an LLM backend for both the root call and sub-calls. This should be configurable per-task via `task.metadata`:

```json
{
    "id": "analyze-codebase-001",
    "prompt": "Find all SQL injection vulnerabilities...",
    "metadata": {
        "rlm": {
            "root_model": "gpt-5",
            "sub_call_model": "gpt-4.1-mini",
            "max_sub_calls": 30,
            "max_depth": 1
        }
    }
}
```

The `task.model` field is ambiguous for RLMs since root and sub-calls may use different models. Placing RLM-specific configuration in `task.metadata` avoids polluting the `TaskDefinition` schema with adapter-specific fields. The adapter reads `task.metadata["rlm"]` and falls back to `task.model` for both root and sub-calls if not specified.

---

## 2. Token Normalization for Recursive Sub-Calls

### The Aggregation Problem

A single RLM `completion()` call produces one root LLM invocation and 5-50 `llm_query()` sub-calls. Each sub-call has its own `input_tokens` and `output_tokens`. The existing `NormalizedTokenUsage` assumes a single call:

```python
@dataclass(frozen=True)
class NormalizedTokenUsage:
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    total_tokens: int = 0  # input + output
```

### Option A: Flat Aggregation

Sum all sub-call tokens into the parent `NormalizedTokenUsage`. The consumer sees one total.

```python
# All sub-call tokens folded into the top-level fields
NormalizedTokenUsage(
    input_tokens=42000,   # root (12000) + sub-calls (30000)
    output_tokens=15000,  # root (8000) + sub-calls (7000)
)
```

| Pro | Con |
|-----|-----|
| Zero changes to `NormalizedTokenUsage` or any consumer | Loses visibility into root vs. sub-call distribution |
| Budget analysis, context pressure, and reports work unchanged | Cannot distinguish "one expensive call" from "many cheap calls" |
| Cost tracking remains accurate (total tokens = total cost) | No way to detect runaway sub-call patterns |

### Option B: Extended Dataclass (Subclass)

```python
@dataclass(frozen=True)
class RlmTokenUsage(NormalizedTokenUsage):
    root_input_tokens: int = 0
    root_output_tokens: int = 0
    sub_call_input_tokens: int = 0
    sub_call_output_tokens: int = 0
    sub_call_count: int = 0
    repl_overhead_tokens: int = 0
```

| Pro | Con |
|-----|-----|
| Full visibility into call structure | `isinstance` checks in consumers may break if they expect exactly `NormalizedTokenUsage` |
| Quality gate can use sub-call fields for RLM-specific signals | Subclass couples the token model to a specific agent type |
| Budget math still works via inherited `total_tokens` | `frozen=True` inheritance with `__post_init__` requires care |

### Option C: Optional Fields on NormalizedTokenUsage

```python
@dataclass(frozen=True)
class NormalizedTokenUsage:
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    total_tokens: int = 0
    # RLM-specific (None for non-RLM agents)
    sub_call_breakdown: dict | None = None
```

Where `sub_call_breakdown` is:

```python
{
    "root_input_tokens": 12000,
    "root_output_tokens": 8000,
    "sub_call_input_tokens": 30000,
    "sub_call_output_tokens": 7000,
    "sub_call_count": 12,
    "repl_overhead_tokens": 800
}
```

| Pro | Con |
|-----|-----|
| No subclass -- single type everywhere | Untyped dict loses static analysis benefits |
| Non-RLM agents set it to `None` -- no noise | Mutable dict inside a frozen dataclass is technically allowed but smells |
| Easy to extend with more fields later | Consumers must null-check before accessing |

**Assessment.** Option A for MVP. Rein's budget and pressure systems need accurate totals, and flat aggregation provides that with zero changes. Option C is the pragmatic next step when RLM-specific quality gate signals need sub-call data. Option B is the cleanest type-theoretically but introduces coupling that the protocol-based adapter architecture is designed to avoid.

---

## 3. Context Pressure for RLMs

### The Dual-Context Problem

Standard agents have one context that grows monotonically. RLMs have two distinct context lifetimes:

1. **Root context**: The RLM's conversation with the REPL. Grows as the model writes code cells, observes outputs, and accumulates state. Subject to the same architectural collapse at 32K-128K tokens as any other LLM interaction (01_theory_mechanism.md, Section 5.2).

2. **Sub-call contexts**: Each `llm_query()` starts fresh. Small, focused prompts (typically 2K-8K tokens). These do not accumulate -- they are independent.

Standard rein pressure: `cumulative_tokens / context_window`. For RLMs, `cumulative_tokens` is ambiguous -- do sub-call tokens count toward the root's pressure?

### Proposed Pressure Model

```python
@dataclass(frozen=True)
class RlmContextPressure:
    # Root context tracking (same semantics as standard ContextPressure)
    root_context_window: int
    root_tokens_used: int           # Tokens in the root REPL conversation
    root_utilization_pct: float     # root_tokens_used / root_context_window * 100
    root_zone: str                  # "green" | "yellow" | "red"

    # Aggregate tracking (for budget/cost purposes)
    total_tokens_all_calls: int     # Root + all sub-calls
    sub_call_count: int

    # Standard ContextPressure fields (for compatibility)
    model_context_window: int       # = root_context_window
    estimated_tokens_used: int      # = root_tokens_used (pressure tracks root only)
    utilization_pct: float          # = root_utilization_pct
    zone: str                       # = root_zone
    measurement: str                # "realtime" | "post_completion"
```

**Key decision: pressure tracks root context only.** Sub-call tokens are billed but do not contribute to context pressure. The root context is what degrades -- sub-calls start fresh. This means an RLM run might consume 200K total tokens (root + sub-calls) while showing only 40% root context pressure. The budget system sees 200K tokens (cost tracking). The pressure system sees 40% (quality tracking). These are measuring different things, and both are correct.

### Zone Actions

Zone actions apply to the root context:

| Root Zone | Action |
|-----------|--------|
| Green | Continue. Sub-calls proceed normally. |
| Yellow | Graceful stop. Wait for the current sub-call to complete, then stop the root RLM. Capture REPL state (all Python variables). Extract any partial results from REPL variables before teardown. |
| Red | Immediate kill. Terminate the RLM process. Capture whatever partial output exists in the stream buffer. REPL state may be lost. |

**Yellow zone recovery.** When the root context hits yellow, the adapter should attempt to extract useful intermediate results from the REPL namespace. In subprocess mode (Option B from Section 1), the wrapper script can serialize all non-internal Python variables to JSON before exiting. In in-process mode, the adapter has direct access to the REPL namespace.

**Sub-call pressure.** Individual sub-calls are unlikely to hit pressure limits (they use 2K-8K tokens against a 128K-200K window). If a sub-call somehow approaches its own model's context window, the RLM framework should handle that internally. Rein does not monitor individual sub-call pressure.

### Measurement Method

For subprocess mode, the wrapper script emits NDJSON events including root token usage after each REPL turn. Rein monitor parses these the same way it parses Claude Code or Codex streams. Sub-call completions are emitted as informational events (for logging) but do not update the pressure calculation.

```jsonl
{"type": "rlm_turn", "root_tokens_used": 15000, "root_context_window": 200000}
{"type": "rlm_sub_call", "index": 0, "input_tokens": 2000, "output_tokens": 500}
{"type": "rlm_turn", "root_tokens_used": 22000, "root_context_window": 200000}
```

The monitor computes pressure from `rlm_turn` events only. `rlm_sub_call` events are buffered for the report.

---

## 4. Hybrid Mode: Standard Agent to RLM Handoff

### Trigger Conditions

When a standard agent (Claude Code, Codex) hits yellow zone, instead of terminating and reporting partial results, rein hands off remaining work to an RLM.

```python
# In runner.py, after yellow zone triggers graceful stop:
if config.hybrid_rlm_handoff and termination_reason == "context_pressure":
    handoff_result = await rlm_handoff(task, sandbox_path, agent_result)
```

**What triggers the switch:**
- The standard agent hits yellow zone (not red -- at red, context is too degraded for a useful state summary)
- `hybrid_rlm_handoff = true` in `rein.toml` (disabled by default)
- The RLM adapter is available (`RlmAdapter().is_available()`)

### Information Transfer

The handoff captures the standard agent's state and constructs an RLM task:

```python
@dataclass(frozen=True)
class HandoffState:
    original_task: TaskDefinition
    git_diff: str                    # What the agent changed
    progress_md: str | None          # PROGRESS.md contents if present
    last_n_messages: list[str]       # Last N stream messages (for context)
    remaining_budget_tokens: int     # Original budget minus consumed
    sandbox_path: Path               # Same sandbox -- code is already there
```

The RLM receives a synthesized prompt:

```
You are continuing work on a coding task. A previous agent made partial progress
before its context became too large.

## Original Task
{original_task.prompt}

## Work Completed (git diff)
{git_diff}

## Progress Notes
{progress_md}

## Remaining Work
Analyze the diff against the original task. Identify what remains undone.
Complete the remaining work.
```

The sandbox is reused -- the standard agent's file modifications are already on disk. The RLM operates on the same filesystem, not a fresh copy. This means the RLM's "context variable" should be loaded from the current sandbox state (post-agent modifications), not the original seed files.

### Quality Gate Evaluation

A hybrid run produces two result segments: the standard agent's partial result and the RLM's completion. The quality gate evaluates the final sandbox state, not the segments independently.

```python
@dataclass(frozen=True)
class HybridResult:
    initial_agent: str               # "claude-code"
    initial_result: AgentResult      # Partial (terminated at yellow)
    handoff_agent: str               # "rlm"
    handoff_result: AgentResult      # RLM completion
    combined_tokens: NormalizedTokenUsage  # Sum of both
    combined_duration: float         # Sum of both durations
```

**Is the RLM run a new round or a continuation?** It is a continuation within the same round. The round mechanism (QUALITY_GATE.md) exists for retry-with-feedback after quality gate failure. The handoff is not a retry -- it is a completion strategy. The quality gate runs once on the final sandbox state after the handoff completes. If the gate fails, a retry round can be dispatched as normal (fresh context, same sandbox, feedback prompt).

**Token accounting.** The combined `NormalizedTokenUsage` sums both segments. The report includes a `handoff` section distinguishing the two:

```json
{
    "result": {
        "handoff": {
            "initial_agent": "claude-code",
            "initial_tokens": {"input": 45000, "output": 12000, "total": 57000},
            "initial_termination_reason": "context_pressure",
            "handoff_agent": "rlm",
            "handoff_tokens": {"input": 30000, "output": 8000, "total": 38000}
        },
        "normalized_tokens": {
            "input_tokens": 75000,
            "output_tokens": 20000,
            "total_tokens": 95000
        }
    }
}
```

### Trade-offs

| Benefit | Cost |
|---------|------|
| Recovers work that would otherwise be lost at yellow zone | Adds a second LLM invocation -- doubles potential cost for the round |
| RLM gets fresh context with only the relevant state | The state summary may lose nuance from the standard agent's reasoning |
| Leverages RLM's strength (large-context analysis of existing code) | Adds complexity to the runner -- a new code path with its own failure modes |
| No changes to the quality gate or round mechanism | The handoff decision is irrevocable -- if the RLM fails, the round fails |

---

## 5. RLM-Informed Task Decomposition

### Pipeline Position

Task decomposition runs as a pre-execution analysis step, before the first round:

```
LOAD task → [DECOMPOSE] → SANDBOX → INVOKE → MONITOR → ...
```

The decomposer is an RLM call that receives the task specification and codebase metadata, and outputs a list of subtask definitions.

### Decomposer Input

```python
@dataclass(frozen=True)
class DecomposerInput:
    task: TaskDefinition
    file_tree: str                   # `find . -type f` output from workspace source
    key_files: dict[str, str]        # Important files (README, config, entry points)
    max_subtasks: int = 5
```

**What the decomposer receives:**
- `task.prompt` -- the full task description
- File tree of the workspace source (for `worktree`/`copy` types) or an empty tree (for `tempdir`)
- Contents of key files: README, main entry points, test files, config files. Selected by heuristic (files matching `README*`, `*main*`, `*config*`, `test_*`, `*_test*`) or explicit list in `task.metadata["decompose_key_files"]`
- Maximum subtask count (to bound decomposition cost)

**What the decomposer does NOT receive:**
- Full file contents of the entire codebase (that is the RLM's job -- it loads the codebase into a REPL variable and analyzes programmatically)
- Previous run history or results

### Decomposer Output

```python
@dataclass(frozen=True)
class SubtaskPlan:
    subtasks: list[TaskDefinition]   # Ordered list of subtasks
    dependency_graph: dict[str, list[str]]  # subtask_id -> [depends_on_ids]
    rationale: str                   # Why this decomposition
    estimated_total_tokens: int      # Rough estimate across all subtasks
```

Each subtask is a full `TaskDefinition` that rein can execute independently:

```json
[
    {
        "id": "refactor-auth-001-sub1",
        "name": "Extract JWT validation into separate module",
        "prompt": "Move JWT validation logic from auth.py into a new jwt_validator.py...",
        "validation_commands": ["python -m pytest tests/test_jwt.py"],
        "files": {},
        "token_budget": 40000
    },
    {
        "id": "refactor-auth-001-sub2",
        "name": "Update auth.py to use new JWT module",
        "prompt": "Refactor auth.py to import and use jwt_validator.py...",
        "validation_commands": ["python -m pytest tests/"],
        "token_budget": 40000
    }
]
```

Subtasks inherit `workspace`, `agent`, `model`, and `effort` from the parent task unless overridden. Each subtask runs in the same sandbox (code from previous subtasks is present) with a fresh agent context.

### Cost/Benefit Analysis

**Cost of decomposition:**
- One RLM call with the codebase as context. For a 500K-token codebase, this costs roughly $0.10-0.50 depending on the model and number of sub-calls.
- Latency: 30-120 seconds for the decomposition call.
- Risk of poor decomposition: subtasks may be too coarse, too fine, or incorrectly ordered. A bad decomposition is worse than no decomposition.

**Benefit of decomposition:**
- Each subtask runs in the green zone (lower context pressure, higher quality).
- Subtask failures are isolated -- a failed subtask does not poison the context for subsequent subtasks.
- The decomposition provides a natural progress checkpoint structure.

**When decomposition is not worth it:**
- Tasks with fewer than ~30K estimated tokens. The decomposition overhead exceeds the context pressure benefit.
- Tasks with unclear scope. If the RLM cannot determine the task boundaries from the spec and codebase, its decomposition will be unreliable (02_benchmarks_evaluation.md, Section 4 on run-to-run variance).
- Tasks requiring tight coordination across subtasks. If every subtask depends on the output of all previous subtasks, the serial overhead eliminates the context pressure benefit.

### Configuration

```toml
[defaults]
decompose = false                    # Opt-in, not default

[decompose]
enabled = false
model = "gpt-5"                      # Model for the decomposer RLM
max_subtasks = 5
max_decompose_budget = 50000         # Token budget for the decomposition call itself
key_file_patterns = ["README*", "*main*", "test_*"]
```

Decomposition can also be requested per-task:

```json
{
    "metadata": {
        "decompose": true,
        "decompose_max_subtasks": 3
    }
}
```

---

## 6. New Quality Gate Signals

### Signal Definitions

```toml
[quality_gate.rlm_health]
enabled = true
required = false
max_sub_calls = 50
max_repl_errors = 5
max_cost_variance_pct = 200
```

This is a built-in signal with special behavior, not a custom command-based signal. It reads from RLM telemetry data (the `rlm_sub_call` NDJSON events or post-completion log parsing).

### Signal Result

```python
SignalResult(
    name="rlm_health",
    status="pass",  # or "fail" or "warn"
    required=False,
    message="12 sub-calls, 0 REPL errors, cost at 85% of median",
    details={
        "sub_call_count": 12,
        "max_sub_calls": 50,
        "recursion_depth": 1,
        "repl_executions": 18,
        "repl_errors": 0,
        "repl_error_rate": 0.0,
        "max_repl_errors": 5,
        "cost_usd": 0.13,
        "median_cost_usd": 0.15,
        "cost_variance_pct": 85,
        "max_cost_variance_pct": 200,
    }
)
```

### Failure Conditions

| Condition | Status | Rationale |
|-----------|--------|-----------|
| `sub_call_count > max_sub_calls` | `fail` | Indicates a redundant verification loop (02_benchmarks_evaluation.md, Appendix B.3 pattern) |
| `repl_errors > max_repl_errors` | `fail` | The model is struggling with REPL code generation -- output quality is suspect |
| `cost_variance_pct > max_cost_variance_pct` | `warn` | Flags outlier cost but does not block acceptance -- the output may still be correct |
| `recursion_depth > 1` | `warn` | Depth > 1 degrades quality significantly (01_theory_mechanism.md, Section 2) |

### Execution Order

`rlm_health` runs after `context_pressure` and before `review`, in the same position as other non-command signals. It is skipped with `status = "skip"` for non-RLM agent runs.

```
build -> tests -> lint -> typecheck -> diff_size -> context_pressure -> rlm_health -> review
```

### Cost Variance Baseline

`cost_variance_pct` requires a historical median. Options:

1. **Per-task median** from previous runs of the same `task.id`. Requires a results database or scanning `results/` directory. Most accurate but requires history.
2. **Per-complexity-class estimate** from `task.metadata["rlm"]["expected_cost_usd"]`. Operator-provided. Simplest.
3. **Disabled by default.** Set `max_cost_variance_pct = 0` to skip variance checking. Only enable after accumulating baseline data.

Option 2 for MVP. The operator sets expected cost in the task definition. The signal warns when actual cost exceeds `expected * (max_cost_variance_pct / 100)`.

---

## 7. Report Format Extension

### RLM Metrics Block

The report's `result` object gains an optional `rlm_metrics` field, present only for RLM agent runs:

```json
{
    "rounds": [
        {
            "result": {
                "agent_name": "rlm",
                "model": "gpt-5",
                "normalized_tokens": {
                    "input_tokens": 42000,
                    "output_tokens": 15000,
                    "total_tokens": 57000
                },
                "rlm_metrics": {
                    "root_tokens": {
                        "input": 12000,
                        "output": 8000,
                        "context_window": 200000,
                        "utilization_pct": 10.0
                    },
                    "sub_calls": [
                        {
                            "index": 0,
                            "model": "gpt-4.1-mini",
                            "input_tokens": 2000,
                            "output_tokens": 500,
                            "prompt_preview": "Classify the following code snippet...",
                            "duration_ms": 1200
                        },
                        {
                            "index": 1,
                            "model": "gpt-4.1-mini",
                            "input_tokens": 2000,
                            "output_tokens": 600,
                            "prompt_preview": "Extract all function signatures...",
                            "duration_ms": 1500
                        }
                    ],
                    "total_sub_calls": 2,
                    "repl_executions": 5,
                    "repl_errors": 0,
                    "repl_error_details": [],
                    "recursion_depth": 1,
                    "total_cost_usd": 0.15,
                    "root_model": "gpt-5",
                    "sub_call_model": "gpt-4.1-mini"
                }
            }
        }
    ]
}
```

### Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `root_tokens` | object | Token usage for the root RLM conversation only |
| `root_tokens.utilization_pct` | float | Root context pressure at completion |
| `sub_calls` | array | Per-sub-call breakdown. Each entry records model, tokens, a truncated prompt preview (first 60 chars), and latency. |
| `total_sub_calls` | int | Length of `sub_calls` array (redundant but convenient for queries) |
| `repl_executions` | int | Total number of Python code cells executed in the REPL |
| `repl_errors` | int | Number of REPL executions that raised Python exceptions |
| `repl_error_details` | array | Exception type and message for each REPL error (for debugging) |
| `recursion_depth` | int | Maximum nesting depth of RLM calls (1 = no nested RLMs) |
| `total_cost_usd` | float | Total API cost across root + all sub-calls |
| `root_model` / `sub_call_model` | string | Model identifiers for root and sub-call LLMs |

### Backward Compatibility

The `rlm_metrics` field is optional. Non-RLM runs omit it entirely. Report consumers that do not know about RLM metrics ignore unknown fields (standard JSON behavior). The `normalized_tokens` field at the top level always contains the flat aggregate (Section 2, Option A), so existing consumers that read `normalized_tokens.total_tokens` get correct totals regardless of whether the run was RLM-powered.

### Hybrid Run Report

When a hybrid handoff occurs (Section 4), the round's result includes both the `handoff` block and `rlm_metrics`:

```json
{
    "result": {
        "agent_name": "claude-code+rlm",
        "handoff": {
            "initial_agent": "claude-code",
            "initial_tokens": {"input": 45000, "output": 12000, "total": 57000},
            "initial_termination_reason": "context_pressure"
        },
        "rlm_metrics": {
            "root_tokens": {"input": 18000, "output": 6000},
            "total_sub_calls": 8,
            "total_cost_usd": 0.09
        },
        "normalized_tokens": {
            "input_tokens": 75000,
            "output_tokens": 20000,
            "total_tokens": 95000
        }
    }
}
```

The `agent_name` uses a compound identifier (`"claude-code+rlm"`) so report analysis can distinguish hybrid runs from pure RLM or pure standard agent runs.

---

## Summary of Design Decisions

| Decision | Recommended for MVP | Alternative |
|----------|-------------------|-------------|
| Adapter architecture | Subprocess wrapper (Option B) | In-process import (Option A) |
| Token normalization | Flat aggregation (Option A) | Optional dict field (Option C) |
| Context pressure target | Root context only | Root + sub-call aggregate |
| Hybrid handoff | Continuation within round | Separate round |
| Task decomposition | Opt-in via config | Auto-detect based on task size |
| RLM health signal | Built-in signal, optional | Custom signal via post-processing script |
| Report extension | Optional `rlm_metrics` field | Separate RLM report file |

Each decision prioritizes backward compatibility and minimal changes to existing rein interfaces. The RLM integration is additive -- it extends rein without modifying the behavior of existing adapters or report consumers.

---

## Sources

- Zhang, Kraska, Khattab. "Recursive Language Models." arXiv:2512.24601, Dec 2025/Jan 2026.
- Wang. "Think, But Don't Overthink: Reproducing Recursive Language Models." arXiv:2603.02615, Mar 2026.
- Rein ARCHITECTURE.md, TOKENS.md, SESSIONS.md, QUALITY_GATE.md, REPORTS.md, TASKS.md, AGENTS.md.
- Deep Dive 01: 01_theory_mechanism.md.
- Deep Dive 02: 02_benchmarks_evaluation.md.
