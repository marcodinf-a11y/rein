# Proposal: Action Frequency Monitoring

**Target:** SESSIONS.md (new section under Context Pressure Monitoring)
**Priority:** Next (when multi-round sessions are common)
**Effort:** Medium

---

## Proposed Section: "Action Frequency Monitoring (Planned)"

The following section is proposed for addition to SESSIONS.md, after the Context Pressure Monitoring section.

---

### Action Frequency Monitoring (Planned)

> **Status: Planned.** Requires Claude Code hook integration (proposal 03) or equivalent tool-call event source.

Context pressure monitoring tracks *how much* of the context window is consumed. Action frequency monitoring tracks *what actions* the agent takes during execution. Together they provide two orthogonal safety signals:

| Dimension | Context Pressure | Action Frequency |
|-----------|-----------------|-----------------|
| **Measures** | Token consumption vs. context window | Tool call counts by type |
| **Detects** | Quality degradation from context growth | Anomalous or dangerous behavior patterns |
| **Triggers** | Zone-based kill (Green/Yellow/Red) | Warning in session report; optional HITL escalation |
| **Mechanism** | Stream parsing (existing) | Tool-call event log (from hooks or stream) |

#### What Gets Counted

The monitor tracks cumulative counts of tool calls by category during a session:

| Category | Example Tool Calls | Why It Matters |
|----------|-------------------|---------------|
| `bash_commands` | Bash, shell execution | High count may indicate looping or brute-force |
| `file_writes` | Write, Edit, file creation | Excessive writes may indicate thrashing |
| `file_reads` | Read, Glob, Grep | Normal — high counts expected |
| `network` | WebFetch, curl, wget | Any count is notable in sandboxed execution |
| `destructive` | rm, chmod, chown, git reset | Any count is notable |

#### Thresholds and Response

Action frequency monitoring does NOT block tool calls. It produces a summary in the session report and optionally flags anomalies.

```json
{
    "action_frequency": {
        "bash_commands": 47,
        "file_writes": 23,
        "file_reads": 89,
        "network": 0,
        "destructive": 2,
        "flags": [
            {
                "category": "bash_commands",
                "count": 47,
                "threshold": 40,
                "message": "High bash command count — possible looping"
            }
        ]
    }
}
```

Default thresholds (configurable in `harness.toml`):

```toml
[action_monitoring]
enabled = false                    # Opt-in — requires hook integration
bash_commands_warn = 40
file_writes_warn = 30
network_warn = 1                   # Any network call in sandboxed execution is notable
destructive_warn = 1               # Any destructive command is notable
```

#### Relationship to Circuit Breakers

Action frequency monitoring is an **observability** mechanism, not an enforcement mechanism. It answers: "what did the agent do?" It does not answer: "should the agent be allowed to do this?"

This is a deliberate design choice. Principal Skinner proposes pre-execution blocking (behavioral circuit breakers). The harness uses sandbox containment instead — let the agent work freely within the sandbox, observe and report what it does, evaluate the output after. Blocking is an arms race (Sondera's known limitation); observation is complete.

If action frequency flags are consistently triggered for a specific task or agent, the operator can:
- Tighten the sandbox (e.g., add network isolation with `--unshare-net`)
- Adjust the task prompt to constrain agent behavior
- Switch to a different agent or model
- Add a custom quality gate signal that fails on dangerous action patterns

#### Data Source

Action events come from one of two sources:

1. **Claude Code hooks** (preferred) — PostToolUse hook logs each tool call with name and input preview. See [proposal 03](03_claude_code_hooks.md).
2. **Stream parsing** (fallback) — Parse tool-use events from the agent's NDJSON/JSONL output stream. Less metadata than hooks but works for all agents.

For agents without hook support (Codex CLI, Gemini CLI), stream parsing is the only option. The monitor extracts tool names from `tool_use` events in the JSONL stream.

---

## Rationale

The Principal Skinner deep dive identifies action-content analysis as the harness's primary safety gap ([synthesis](../00_synthesis.md) §4.4). The harness monitors context pressure (resource-based) but not action patterns (behavior-based). Principal Skinner proposes full behavioral circuit breakers; the critical analysis ([06_critical_analysis.md](../06_critical_analysis.md)) concludes this is over-engineering for local development.

Action frequency monitoring is the pragmatic middle ground:

1. **No policy engine required.** Simple counters, not Cedar/OPA rule evaluation.
2. **No execution blocking.** Sandbox containment handles safety; monitoring handles visibility.
3. **Minimal implementation.** Count tool calls by category, compare against thresholds, include in report.
4. **Bridges the gap.** The harness goes from knowing "the agent used 60% of context" to knowing "the agent used 60% of context AND ran 47 bash commands, 2 destructive operations, and 0 network calls."

This is classified as "Next" because it depends on either hook integration (proposal 03) or stream-parsing enhancement, and because the value increases with multi-round sessions where behavioral patterns are more pronounced.

## Source References

- [Principal Skinner Synthesis](../00_synthesis.md) — Section 4.4 (action frequency monitoring)
- [Circuit Breakers](../03_circuit_breakers.md) — Section 4 (harness gap: no action-content analysis)
- [Tool-Use Control](../02_tool_use_control.md) — Section 6 (observation vs. enforcement)
- [Critical Analysis](../06_critical_analysis.md) — Section 3 (over-engineering assessment)
- SESSIONS.md — Context Pressure Monitoring (the existing resource-based monitoring this complements)
