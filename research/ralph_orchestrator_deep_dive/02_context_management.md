# Ralph Orchestrator: Context Window Management

**Deep Dive Document 02 | March 2026**

How ralph-orchestrator tracks, manages, and rotates context. Compared against Rein's green/yellow/red zone model.

---

## 1. How It Tracks Token Usage

**Ralph does not track token usage in real-time.** It does not monitor how full the agent's context window is during execution. There is no tiktoken, no token counting, no context pressure calculation.

What it does track:
- **Post-hoc cost:** Parses the agent's output stream for cost/token metadata after each iteration completes. For Claude, this comes from `--output-format stream-json` `Result` events containing `total_cost_usd`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`.
- **Cumulative cost:** `LoopState.cumulative_cost` accumulates USD across iterations for the `max_cost_usd` check.

**This is a fundamental difference from rein.** Ralph does not know, and cannot know, how close the agent is to context degradation during a given iteration. It avoids the problem entirely by keeping iterations short enough that context exhaustion within a single iteration is unlikely.

Source: `stream_parser.rs`, `execution_outcome.rs` in ralph-core.

---

## 2. Token Thresholds and Actions

**There are no token-based thresholds.** Ralph has no equivalent to Rein's green/yellow/red zones. It does not trigger graceful stops, kills, or warnings based on token usage.

Instead, ralph-orchestrator uses these termination conditions:

| Condition | Default | Action |
|-----------|---------|--------|
| `max_iterations` | 100 | Terminate loop |
| `max_runtime_seconds` | 14,400 (4h) | Terminate loop |
| `max_cost_usd` | None | Terminate loop (if set) |
| `max_consecutive_failures` | 5 | Terminate loop |
| Stale loop (same topic 3×) | Always on | Terminate loop |
| Thrashing (abandoned tasks redispatched 3×) | Always on | Terminate loop |

None of these are context-window-aware. They are resource caps (time, money, iterations) and behavioral detectors (stagnation), not quality-risk detectors.

Source: `config.rs`, `loop_state.rs`, `termination.rs` in ralph-core.

---

## 3. Comparison with Rein's Zone Model

| Dimension | ralph-orchestrator | Rein |
|-----------|-------------------|----------------|
| **Token tracking** | Post-hoc from backend stream | Real-time stream parsing |
| **Context pressure** | Not computed | `utilization_pct = tokens / context_window` |
| **Quality risk zones** | None | Green (0-60%), Yellow (60-80%), Red (80%+) |
| **Mid-iteration intervention** | Not possible | Graceful stop (Yellow), immediate kill (Red) |
| **Cost tracking** | Cumulative USD from backend | Token budget with utilization status |
| **Measurement method** | `post_completion` only | `realtime`, `post_completion`, `unmonitored` |
| **Cache awareness** | Tracks cache tokens (Claude) | Normalizes cache across agents, excludes from budget |

**Rein can intervene during an iteration; ralph cannot.** If an agent enters a death spiral within a single iteration (e.g., requesting enormous context, hitting compaction), ralph will not notice until the iteration completes (or the agent crashes). Rein detects this in real-time and kills the process before quality degrades.

---

## 4. Context Rotation

Ralph's context rotation is identical in principle to bare Ralph: **kill the process, start fresh.**

Each iteration:
1. `EventLoop.build_prompt()` assembles a fresh prompt from scratchpad, tasks, skills, and hat instructions
2. A new subprocess is spawned via PTY
3. The agent runs with zero conversation history from prior iterations
4. The agent exits (or is killed on timeout)
5. Output is captured, events are read from JSONL
6. Loop continues

**What carries forward between iterations:**
- `.ralph/agent/scratchpad.md` — agent's working notes (capped at ~16K chars)
- `.ralph/events-*.jsonl` — event history (for pub/sub routing, not re-injected as context)
- `.ralph/tasks.jsonl` — task status
- `.ralph/memories/` — persistent knowledge
- Git history — code changes

**What does not carry forward:**
- Conversation history
- In-context tool outputs
- Reasoning chains

This is the same "deterministic allocation" pattern identified in the Ralph Wiggum deep dive ([03_validated_patterns.md](../ralph_wiggum_deep_dive/03_validated_patterns.md)): every iteration starts with identical structural context (scratchpad + tasks + instructions), ensuring specs survive compaction.

---

## 5. Scratchpad: The Context Bridge

The scratchpad (`.ralph/agent/scratchpad.md`) is ralph-orchestrator's key innovation over bare Ralph:

- **Read at start:** Injected into the prompt each iteration
- **Written during iteration:** Agent appends notes, decisions, blockers
- **Budget-capped:** ~16,000 characters (tail-retained, FIFO eviction)
- **Structural awareness:** Section headings are preserved even when content is trimmed

This is functionally equivalent to Rein's seed files + LEARNINGS.md concept, but with automatic budget management. Rein carries forward artifacts via `files` in task JSON; ralph carries forward operational notes via scratchpad.

**Key difference:** The scratchpad is a single shared document that grows and is trimmed. Rein's seed files are per-task and explicitly defined by the operator. Ralph's approach is more autonomous (agent decides what to write); Rein's is more controlled (operator decides what to seed).

---

## 6. Compaction Awareness

**Ralph does not detect or prevent compaction events.** Since each iteration is a fresh subprocess with a fresh context, compaction within the underlying agent's session is the agent's own problem — and because iterations are designed to be short (single-task), the risk of hitting compaction within one iteration is low.

However, there is no safeguard if:
- The prompt + scratchpad + skills exceed the agent's context window
- The agent's own tool use generates massive context within one iteration
- The agent enters a recursive exploration that fills its window

Rein addresses all three via real-time pressure monitoring and zone-based intervention.

Source: no compaction-related code found in ralph-core or ralph-adapters.

---

## Sources

- github.com/mikeyobrien/ralph-orchestrator (v2.7.0)
- Source files: `stream_parser.rs`, `config.rs`, `loop_state.rs`, `scratchpad.rs`, `event_loop.rs`
- ARCHITECTURE.md, TOKENS.md (rein)
- research/ralph_wiggum_deep_dive/03_validated_patterns.md
