# RLM Implementation Architecture: Rein Adapter Compatibility Analysis

## Overview

The RLM library (`pip install rlms`) is a Python library, not a CLI tool. Rein was designed around subprocess invocation of CLI agents (Claude Code, Codex CLI, Gemini CLI). This document analyzes the architectural friction between RLM's in-process library design and Rein's subprocess-oriented adapter protocol, covering class hierarchy mapping, sandbox overlap, token tracking, JSONL compatibility, execution model tradeoffs, and security implications.

---

## 1. Class Hierarchy to Adapter Protocol Mapping

Rein adapter protocol is minimal:

```python
@runtime_checkable
class AgentAdapter(Protocol):
    @property
    def name(self) -> str: ...
    def is_available(self) -> bool: ...
    async def run(self, task: TaskDefinition, sandbox_path: Path) -> AgentResult: ...
```

Every existing adapter (claude.py, codex.py, gemini.py) implements `run()` by calling `asyncio.create_subprocess_exec` against a CLI binary, reading stdout as NDJSON/JSONL, and normalizing the output into `AgentResult`. RLM has no CLI binary. The `RLM` class is instantiated in Python and called via `rlm.completion()`.

Three integration strategies exist, each with distinct tradeoffs:

### Option A: CLI Wrapper Script

Write a thin Python script (e.g., `rlm_runner.py`) that the adapter invokes as a subprocess. The script imports `rlm`, runs `completion()`, and prints results as JSONL to stdout. The adapter reads this stream identically to how it reads Codex output.

```
rein runner.py
  -> asyncio.create_subprocess_exec("python", "rlm_runner.py", ...)
    -> rlm_runner.py imports rlm, calls completion()
    -> prints JSONL to stdout
  -> adapter reads stdout, normalizes tokens
```

**Advantages:** Maintains Rein's subprocess isolation model. The adapter code looks nearly identical to existing adapters. Process-level fault isolation -- if RLM crashes or hangs, rein can SIGKILL it. Context pressure monitoring works via the same stream-parsing infrastructure.

**Disadvantages:** Serialization overhead for result data. Cannot access RLM's in-memory logger or Python objects directly. The wrapper script becomes a maintenance artifact that must be kept in sync with the RLM API. Startup cost of a new Python process plus `import rlm` on every invocation.

### Option B: In-Process Import

The adapter imports `rlm` directly and calls `completion()` within rein process. No subprocess boundary.

```python
class RlmAdapter:
    async def run(self, task: TaskDefinition, sandbox_path: Path) -> AgentResult:
        rlm_instance = RLM(environment="docker", backend="openai", ...)
        result = await asyncio.to_thread(rlm_instance.completion, prompt=task.prompt)
        return self._normalize(result)
```

**Advantages:** Zero serialization overhead. Direct access to the `result` object and its attributes. Direct access to RLMLogger's in-memory data. Simpler deployment -- no wrapper script to maintain.

**Disadvantages:** Breaks the subprocess isolation model that all other adapters follow. If RLM hangs, rein cannot SIGKILL it without killing itself. If RLM's `exec()` (LocalREPL) executes malicious code, it runs in rein process with full access. Context pressure monitoring cannot use the stream-parsing infrastructure -- would need a callback or polling mechanism instead. `asyncio.to_thread` moves the blocking call off the event loop, but the RLM still shares the process address space.

### Option C: Hybrid

Import RLM in-process but run it in a forked child process using `multiprocessing`, communicating via a pipe or queue. This preserves process isolation without requiring a separate script file.

**Advantages:** Process isolation without a standalone wrapper script. Can use `multiprocessing.Process` with a target function, avoiding the CLI serialization overhead while still being killable. Pipe-based communication allows structured Python objects.

**Disadvantages:** `multiprocessing` adds complexity around pickling, error propagation, and cleanup. Forked processes inherit the parent's memory, which may include sensitive state. Still cannot use Rein's NDJSON stream parser without additional plumbing.

### Recommendation

Option A (CLI wrapper) is architecturally cleanest for rein. Rein's entire monitoring, termination, and capture subsystem assumes a subprocess emitting NDJSON/JSONL. Deviating from this for one adapter adds conditional logic throughout the pipeline. The serialization overhead is negligible compared to LLM API latency. The wrapper script is small -- under 100 lines -- and can be bundled as a package entry point.

---

## 2. Sandbox Overlap

RLM and rein both have sandbox concepts, but they serve fundamentally different purposes and operate at different layers.

**Rein sandbox** (sandbox.py): Creates an isolated *filesystem workspace* for the agent to operate in. Modes are tempdir, worktree, and copy. The sandbox is the agent's `cwd` -- it contains seed files, receives the agent's edits, and is the source for diff capture and artifact collection. It is workspace isolation.

**RLM sandbox** (Environment): Creates an isolated *code execution environment* for the REPL. The REPL is where the LLM's generated Python code runs -- `exec()` calls, `llm_query()` sub-calls, variable storage. It is compute isolation.

These are orthogonal. Rein sandbox answers "where does the agent's work happen on disk?" The RLM sandbox answers "where does the LLM-generated Python code execute?" They can coexist without conflict:

```
rein sandbox (tempdir/worktree/copy)
  └── contains task files, receives edits
  └── agent cwd

RLM environment (LocalREPL/DockerREPL/Modal/E2B)
  └── executes LLM-generated Python code
  └── hosts the `context` variable and `llm_query()`
```

The only interaction point is if RLM-generated code attempts to read or write files in rein sandbox directory. With LocalREPL, this is unrestricted -- the `exec()` call shares the process filesystem. With DockerREPL, the sandbox directory would need to be mounted as a volume. With cloud sandboxes, there is no filesystem overlap at all.

For coding tasks where RLM needs to interact with repo files (not just process long text), rein sandbox path would need to be made available to the RLM environment. This is a configuration concern, not an architectural conflict.

---

## 3. Token Tracking Compatibility

Rein normalizes all token usage into `NormalizedTokenUsage`:

```python
@dataclass(frozen=True)
class NormalizedTokenUsage:
    input_tokens: int       # Total input (inclusive of cache)
    output_tokens: int
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    total_tokens: int = 0   # Computed: input + output
```

### What RLM Exposes

The RLM `completion()` call returns a result object with `.response` confirmed. Based on the library's architecture, the underlying LLM client (OpenAI, Anthropic, etc.) returns token usage per API call. The critical question is whether `rlm.completion()` aggregates these across all sub-calls and exposes them on the result object.

RLMLogger tracks execution events including sub-calls. If the logger captures per-call token usage, the adapter could extract aggregated totals from logger data even if the result object does not surface them directly.

### The Aggregation Problem

A single `rlm.completion()` call may invoke 5-50+ LLM API calls internally:
- The root LM call (system prompt + REPL interaction loop, potentially multi-turn)
- Each `llm_query()` sub-call (one LLM API call per sub-query)
- Each REPL iteration that feeds observations back to the LM

Each of these API calls has its own `input_tokens` and `output_tokens`. Rein needs a single aggregated `NormalizedTokenUsage` for the entire `rlm.completion()` invocation.

If RLM exposes per-call usage via its logger, the adapter sums them:

```python
def _normalize_rlm(logger_data) -> NormalizedTokenUsage:
    total_input = sum(call.input_tokens for call in logger_data.calls)
    total_output = sum(call.output_tokens for call in logger_data.calls)
    # Cache fields depend on the underlying LLM client
    return NormalizedTokenUsage(
        input_tokens=total_input,
        output_tokens=total_output,
    )
```

If RLM does not expose per-call usage, the adapter would need to wrap the underlying LLM client with a proxy that intercepts and accumulates token counts. This is invasive but feasible since RLM accepts configurable backends.

### Context Pressure Implications

Rein computes context pressure as `estimated_tokens_used / model_context_window`. For CLI agents, this tracks cumulative tokens across turns within a single conversation. For RLM, the semantics differ: each sub-call is an independent LLM invocation with its own context window. The root call accumulates REPL history in its context, but sub-calls start fresh.

This means context pressure for RLM is best measured on the root call's context growth (REPL interaction history accumulating across iterations), not on the sum of all sub-call tokens. The sum of all tokens is relevant for cost tracking but not for context rot risk -- sub-calls have their own fresh windows.

---

## 4. JSONL Logger Compatibility

RLMLogger supports JSONL disk output. Rein's context pressure monitor reads NDJSON/JSONL from agent stdout line by line. The question is whether these formats are compatible.

### Rein Stream Parser Expectations

Rein expects specific event schemas per agent. For Codex: `turn.completed` with `usage` fields. For Claude: `message_start` and `message_delta` with `usage` fields. There is no generic event schema -- each adapter knows its agent's format.

### RLMLogger Event Model

RLMLogger likely emits events such as: REPL code execution, observation output, sub-call invocation, sub-call completion, final answer. These do not map to any existing rein event schema.

### Integration Path

Under Option A (CLI wrapper), the wrapper script would need to either:

1. **Translate RLMLogger events to a rein-compatible schema.** Define an `rlm`-specific event format that the `RlmAdapter` knows how to parse. Events like `{"type": "subcall.completed", "usage": {"input_tokens": N, "output_tokens": M}}` emitted to stdout as NDJSON.

2. **Emit only final results.** Skip mid-run streaming entirely. The wrapper runs `completion()` to completion, then prints a single JSON result line. This means `measurement = "post_completion"` -- no real-time context pressure monitoring, similar to Gemini's current limitation.

Option 2 is simpler and adequate for an initial integration. Real-time monitoring of RLM sub-calls is a future enhancement that would require hooking into RLM's internal call loop -- either via logger callbacks or by patching the client's API call method.

---

## 5. Subprocess vs. In-Process: Architectural Analysis

Rein's subprocess model provides three guarantees that in-process execution cannot:

1. **Fault isolation.** A crashed or hung agent does not bring down rein. Rein detects exit codes, timeouts, and can SIGKILL.
2. **Resource control.** Rein controls the subprocess's environment variables, working directory, and can apply OS-level resource limits (ulimit, cgroups).
3. **Clean termination.** The subprocess termination procedure (SIGTERM -> 5s -> SIGKILL) provides deterministic cleanup with known semantics.

In-process RLM execution loses all three. A hung `completion()` call blocks the event loop thread (even with `asyncio.to_thread`, the thread itself is blocked). A crash in `exec()` could corrupt Rein's memory. There is no mechanism to forcibly terminate a Python function call without killing the entire process.

The performance argument for in-process execution is weak. The dominant latency in any RLM call is LLM API round-trips (seconds to minutes per call, multiplied by 5-50 calls). Python process startup (~200ms) and import time (~500ms for `rlm`) are negligible in comparison.

---

## 6. Security Analysis

RLM's execution model introduces security concerns that do not exist with Rein's current CLI agents.

### Threat Model

Current rein agents (Claude Code, Codex, Gemini) execute as CLI subprocesses. They make their own tool calls internally (file edits, shell commands) but rein only interacts with their stdout stream. Rein process itself never executes LLM-generated code.

RLM fundamentally changes this. The REPL executes arbitrary Python code generated by the LLM. The `context` variable may contain untrusted input (user-provided documents, web content, code from unknown repos). Prompt injection via the context could cause the LLM to generate malicious code.

### Per-Environment Risk Assessment

| Environment | Risk Level | Attack Surface |
|---|---|---|
| LocalREPL | Critical | `exec()` in host process. Full filesystem, network, and process access. Can read env vars (API keys), modify files, exfiltrate data. |
| DockerREPL | Moderate | Containerized but requires Docker daemon socket. Container escape vulnerabilities exist. Network access unless explicitly restricted. |
| Modal/E2B/Daytona | Low | True isolation. Code runs in ephemeral cloud sandboxes. No access to host filesystem or network (unless configured). Adds latency (cold start) and cost. |

### LocalREPL vs. Rein Subprocess Model

Rein runs agents as subprocesses -- but those agents (Claude Code, Codex) also execute LLM-generated code internally. The difference is that those agents have their own sandboxing:
- Codex defaults to `read-only` sandbox mode
- Claude Code has `--dangerously-skip-permissions` (which rein uses, but the agent still controls its own execution)
- Gemini has `--yolo` approval mode

With LocalREPL, there is no intermediary agent applying its own safety checks. The LLM-generated code runs via `exec()` with no sandbox, no approval, no restriction.

### Mitigation for Rein Integration

The adapter should enforce `environment="docker"` as the minimum. LocalREPL should be unavailable when running under rein, regardless of user configuration. The adapter constructor should reject or override `environment="local"`:

```python
class RlmAdapter:
    def __init__(self, environment: str = "docker", **kwargs):
        if environment == "local":
            raise ValueError(
                "LocalREPL is not permitted under rein. "
                "Use 'docker', 'modal', 'e2b', or 'daytona'."
            )
        self._environment = environment
```

For maximum security, cloud sandboxes (Modal, E2B) are preferable but add latency per REPL execution step. DockerREPL is the practical middle ground -- containerized execution with acceptable latency.

---

## 7. API Surface and Integration Gaps

### Known API Surface

```python
from rlm import RLM, RLMLogger

logger = RLMLogger(log_dir="./logs")
rlm = RLM(
    environment="local",
    backend="openai",
    backend_kwargs={"model": "gpt-5-nano"},
    verbose=True,
)
result = rlm.completion(prompt="...", logger=logger)
print(result.response)
```

### Open Questions

**Result object contents.** Beyond `.response`, does the result expose `.usage`, `.cost`, `.num_subcalls`, `.total_tokens`? If not, token tracking requires extracting data from the logger or instrumenting the client.

**Blocking vs. async.** `rlm.completion()` is almost certainly synchronous and blocking. It loops internally: call LLM -> execute code -> observe -> repeat. There is no documented streaming or async API. The adapter must wrap it in `asyncio.to_thread` (in-process) or run it in a subprocess (wrapper script).

**Callback hooks.** Can the adapter register callbacks for mid-execution events (sub-call start, sub-call end, code execution)? If RLMLogger supports event callbacks rather than just file output, real-time monitoring becomes possible without stdout parsing.

**Backend configuration.** `backend_kwargs` passes through to the LLM client. For Anthropic backend, this would include `model`, `max_tokens`, etc. Rein's task-level `model` and `effort` fields need to be mapped to `backend_kwargs`. This mapping is backend-specific -- OpenAI's `model` parameter differs from Anthropic's.

**Environment lifecycle.** Does the RLM class manage environment setup/teardown (Docker container creation, Modal function deployment)? If so, rein should create the RLM instance once and reuse it across calls to amortize setup cost. If not, each `completion()` call pays the environment startup penalty.

---

## Summary of Integration Friction Points

| Friction Point | Severity | Resolution |
|---|---|---|
| No CLI binary -- library only | High | CLI wrapper script (Option A) |
| Token usage aggregation across sub-calls | High | Logger extraction or client instrumentation |
| LocalREPL security risk | Critical | Enforce DockerREPL minimum |
| No NDJSON stream output | Medium | Post-completion measurement initially |
| Blocking API, no async | Low | `asyncio.to_thread` or subprocess |
| Context pressure semantics differ | Medium | Track root call context growth, not sub-call sum |
| Unknown result object schema | Medium | Requires empirical investigation of `rlm` package |

The cleanest integration path is: CLI wrapper script that imports `rlm`, enforces DockerREPL, runs `completion()`, extracts token data from the logger, and emits a single JSONL result line to stdout. The adapter parses this identically to how it parses Codex or Gemini output. Context pressure monitoring starts as `post_completion` and can be upgraded to `realtime` if RLMLogger supports event callbacks that the wrapper can relay as NDJSON events.
