# Proposal: Agent Git Identity Per Session

**Target:** SESSIONS.md (enhancement to Harness Wrap-Up Protocol)
**Priority:** Now
**Effort:** Low

---

## Proposed Enhancement: Agent-Specific Git Author

The following describes a proposed enhancement to the sandbox setup and wrap-up protocol in SESSIONS.md.

---

### Agent Git Identity

Every sandbox session configures a git author identity that distinguishes agent-authored commits from human-authored commits. This enables post-mortem attribution: when reviewing git history, it is immediately clear which commits were produced by which agent, model, and effort level.

#### Configuration

When the harness creates a sandbox, it sets git config for that sandbox directory:

```bash
git -C "$SANDBOX_PATH" config user.name "claude-code/opus/high"
git -C "$SANDBOX_PATH" config user.email "agent@harness.local"
```

The `user.name` follows the format `{agent}/{model}/{effort}`:

| Agent | Model | Effort | Git Author |
|-------|-------|--------|------------|
| Claude Code | opus | high | `claude-code/opus/high` |
| Claude Code | sonnet | medium | `claude-code/sonnet/medium` |
| Codex CLI | o3 | high | `codex-cli/o3/high` |
| Gemini CLI | pro | high | `gemini-cli/pro/high` |

The email address is a fixed constant (`agent@harness.local`) — it is not a real email. Its purpose is to satisfy git's requirement for a commit email and to make agent commits greppable:

```bash
# Find all agent-authored commits
git log --author="agent@harness.local"

# Find commits by a specific agent/model
git log --author="claude-code/opus"
```

#### Scope

- The git config is set per-sandbox directory (`--local`), not globally. It does not affect the operator's global git config.
- The config applies to both agent-authored commits (during execution) and harness-authored commits (during wrap-up).
- For harness wrap-up commits (auto-commit at termination), the author is the same agent identity — the wrap-up captures the agent's work.

#### Report Integration

The `result` section of the structured JSON report already contains `agent_name`, `model`, and `effort`. The git author identity is derived from these same fields, ensuring consistency between the report and the git history.

No new report fields are needed. The existing `diff` field in the report captures the git diff, and the commit metadata (including author) is available via standard git commands in the sandbox.

---

## Rationale

Principal Skinner's agent identity argument is sound: "You can only debug what you can identify" (securetrajectories.substack.com). OWASP ASI03 (Identity & Privilege Abuse) recommends isolated identities per agent with task-scoped permissions.

Currently, the harness's sandbox commits use the operator's git config. This means:

1. **Git history doesn't distinguish agent work from human work.** In a worktree sandbox where changes are merged back, agent-authored commits appear as if the human wrote them.

2. **Post-mortem analysis requires cross-referencing reports.** To determine which agent produced a commit, the operator must match timestamps between git log and JSON reports. With agent-specific git authors, `git log --author` provides this directly.

3. **Multi-agent workflows will need this.** When the harness supports running different agents on different tasks (Claude for implementation, Codex for testing), git attribution becomes essential for understanding which agent contributed what.

The implementation cost is near-zero: two `git config` calls during sandbox setup. The Principal Skinner deep dive correctly identifies this as the highest-value, lowest-cost improvement from the entire safety analysis.

What is explicitly NOT proposed:
- **Per-agent SSH keys** — enterprise-scale, irrelevant for local development
- **Per-agent service accounts** — requires identity infrastructure the harness doesn't need
- **Per-agent API credentials** — agents use the operator's API keys; credential isolation is out of scope

## Source References

- [Principal Skinner Synthesis](../00_synthesis.md) — Section 4.2 (agent git identity)
- [Agent Identity](../04_agent_identity.md) — Section 4 (harness comparison), Section 5 (low-cost fix)
- [OWASP ASI03](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) — Identity & Privilege Abuse
- SESSIONS.md — Harness Wrap-Up Protocol (where auto-commits occur)
- ARCHITECTURE.md — Execution Isolation (sandbox setup)
