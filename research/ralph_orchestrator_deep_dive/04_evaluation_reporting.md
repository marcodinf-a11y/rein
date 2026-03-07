# Ralph Orchestrator: Evaluation & Reporting

**Deep Dive Document 04 | March 2026**

What data ralph-orchestrator captures, how it reports results, and how this compares to Rein's structured evaluation.

---

## 1. Data Captured Per Iteration

Each iteration records:

| Data Point | Source | Notes |
|------------|--------|-------|
| Iteration number | Internal counter (1-indexed) | Sequential |
| Active hat ID + name | Event routing | Which persona executed |
| Agent output (full text) | PTY/CLI stream | Complete stdout |
| Success/failure | Process exit code | Boolean |
| Cost (USD) | Backend stream metadata | Claude `Result` event |
| Input tokens | Backend stream | Claude `Assistant` events |
| Output tokens | Backend stream | Claude `Assistant` events |
| Cache read/write tokens | Backend stream | Claude-specific |
| Duration | Internal timer | Wall-clock |
| Events emitted | JSONL file read | Structured topics |
| Completion signal | Event check | Whether `LOOP_COMPLETE` fired |
| Consecutive failure count | Loop state | Running tally |
| Cumulative cost | Loop state | Sum across all iterations |

**For Claude backends**, the data is comprehensive. For other backends (Gemini, Codex, etc.), token-level data may not be available — ralph depends on what each CLI reports.

Source: `ExecutionOutcome` struct, `LoopState` in ralph-core.

---

## 2. Report Format

On termination, `SummaryWriter` generates `.ralph/agent/summary.md`:

```markdown
# Loop Summary
**Status:** Completed successfully | Stopped: max iterations | Failed: ...
**Iterations:** 12
**Duration:** 23m 45s
**Est. cost:** $1.50

## Tasks
- [x] Implement auth module
- [ ] Add rate limiting (abandoned: blocked 3x)

## Events
- 12 total events
- 6 build.task
- 5 build.done

## Final Commit
abc1234: feat(auth): add JWT validation

## Landing
- Auto-committed: Yes (abc1234)
- Handoff: `.ralph/agent/handoff.md`
- Open tasks: 2
```

**Format:** Markdown only. Not JSON, not NDJSON. Human-readable, not machine-parseable.

Additionally, events are logged to `.ralph/agent/events.jsonl` in structured format. The RPC API emits `IterationStart` and `IterationEnd` events with telemetry for the web dashboard.

Source: `summary_writer.rs`, `event_logger.rs` in ralph-core.

---

## 3. Normalized Comparison Across Runs

**Not supported.** Ralph does not produce normalized metrics that allow comparing one run to another on the same task. There is no:
- Normalized token usage (each backend reports differently)
- Score per run (only pass/fail per iteration gate)
- Cross-run aggregation
- Statistical analysis (mean, variance across attempts)

Each run produces a standalone summary. Comparing runs requires manual inspection of summaries.

Source: no normalization or comparison logic found in ralph-core.

---

## 4. Machine-Readable Output

| Output | Format | Machine-Readable |
|--------|--------|-----------------|
| Summary | Markdown | No |
| Events | JSONL | Yes |
| Tasks | JSONL | Yes |
| Scratchpad | Markdown | No |
| Handoff | Markdown | No |
| RPC telemetry | JSON (over RPC) | Yes (dashboard only) |

The JSONL event log is the most machine-readable artifact. Each line is a JSON object with `topic`, `payload`, `timestamp`, and `hat_id`. This could be parsed for trend analysis, but ralph provides no tooling for doing so.

---

## 5. Comparison with Rein

| Dimension | ralph-orchestrator | Rein |
|-----------|-------------------|----------------|
| **Report format** | Markdown summary | Structured JSON |
| **Token accounting** | Per-iteration from backend | Normalized `NormalizedTokenUsage` across agents |
| **Cost tracking** | Cumulative USD | Token budget with utilization % |
| **Quality score** | None (pass/fail gates only) | Binary score (0.0/1.0) from validation commands |
| **Context pressure** | Not tracked | Zone, utilization %, measurement method |
| **Diff capture** | Final commit hash | Full git diff |
| **Cross-run comparison** | Not supported | Structured reports enable comparison |
| **Machine-readable** | JSONL events only | Full JSON report |
| **Cache efficiency** | Raw cache tokens (Claude only) | Cache efficiency ratio across agents |
| **Termination metadata** | Status + reason in summary | Full termination metrics (zone, turns, signal, duration) |

**Key gap:** Ralph captures data but does not structure it for analysis. Rein's JSON reports are designed for programmatic comparison — you can diff two runs, aggregate across tasks, compute trends. Ralph's markdown summaries require human reading.

**Key strength:** Ralph's JSONL event log is a structured event stream that could be post-processed. Rein does not have an equivalent event bus — it captures snapshots (before/after), not a continuous event stream.

---

## 6. What Ralph Could Learn from Rein

1. **Structured JSON reports** — machine-readable summaries for CI/CD integration and trend analysis
2. **Normalized token accounting** — consistent metrics regardless of backend
3. **Quality scoring** — a formal score per run, not just "did it complete"
4. **Context pressure tracking** — utilization % as a quality signal

## 7. What Rein Could Learn from Ralph

1. **JSONL event stream** — continuous structured events during execution, not just before/after snapshots
2. **Per-iteration telemetry** — rein runs one session per task; ralph's per-iteration granularity reveals patterns within a task
3. **Task-level progress tracking** — ralph tracks which tasks are done, abandoned, blocked — rein only knows pass/fail at the end

---

## Sources

- github.com/mikeyobrien/ralph-orchestrator (v2.7.0)
- Source files: `summary_writer.rs`, `event_logger.rs`, `execution_outcome.rs`, `loop_state.rs`
- TOKENS.md, ARCHITECTURE.md (rein)
