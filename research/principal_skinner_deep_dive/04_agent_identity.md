# Principal Skinner: Agent Identity & Attribution

**Deep Dive Document 04 | March 2026**

How Principal Skinner proposes agent identity, what it actually implements, and whether rein has a genuine gap worth fixing.

---

## 1. Principal Skinner's Proposal

Principal Skinner argues that agents must have distinct identities: unique SSH keys, per-agent service accounts, and Agent IDs attached to every operation. The rationale is forensic: "You can only debug what you can identify." If an agent deletes a file, the post-mortem must distinguish between operator error and agentic misalignment.

The "Anthropic Attack" article extends this to full lifecycle observability: "every decision, every tool call, every observation" logged with agent identity, producing immutable audit trails. The three-phase model (Crawl/Walk/Run) places identity in the Walk phase — it is a prerequisite for enforcement, not an afterthought.

**Implementation status:** Conceptual only. No code, no tooling, no reference implementation. The proposal describes standard DevOps identity management (SSH keys, service accounts, audit logs) applied to agents. This is repackaging, not invention.

---

## 2. The Attribution Problem Is Real

Despite the lack of implementation, the underlying problem is genuine. When a Ralph Loop commits code, git attributes it to the human developer. A `git log` six months later reveals no distinction between human-authored and agent-authored changes. This creates three concrete problems:

1. **Post-mortem confusion.** A bug introduced by an agent looks like a bug introduced by the developer. Debugging requires understanding the commit's provenance — was this a deliberate design choice or an agent's best guess?

2. **Compliance liability.** In regulated environments, the question "who wrote this code?" has legal weight. "My AI agent did" is not currently distinguishable from "I did" in git history.

3. **Quality signal loss.** Agent-authored code may have different failure distributions than human-authored code. Without attribution, you cannot analyze these distributions separately.

OWASP ASI03 (Identity & Privilege Abuse) maps directly: isolated identities per agent, short-lived credentials, task-scoped permissions. The standard explicitly warns against "unintended reuse or escalation of credentials across agents."

---

## 3. What Rein Already Does

Rein tracks identity at the **report level**, not the **commit level**:

| Data Point | Tracked | Where |
|-----------|---------|-------|
| Agent name (claude-code, codex-cli, gemini-cli) | Yes | Session report JSON |
| Model (sonnet, opus, flash, pro) | Yes | Session report JSON |
| Effort level (low, medium, high) | Yes | Session report JSON |
| Task ID | Yes | Session report JSON |
| Token consumption (normalized) | Yes | Session report JSON |
| Git diff | Yes | Session report JSON |
| Git commit author | No | Uses operator's git config |
| Per-agent SSH key | No | Not relevant for local dev |
| Per-agent service account | No | Not relevant for local dev |

Rein knows *which agent produced which result* — but this knowledge lives in structured JSON reports, not in git history. A `git log` in the sandbox shows the operator's name on every commit, regardless of which agent wrote the code.

---

## 4. The Low-Cost Fix

Configuring git author per session is trivial and high-value. Rein already controls the subprocess environment. Adding two environment variables costs nothing:

```
git -c user.name="claude-code/opus/high" -c user.email="agent@rein.local" commit ...
```

Or, equivalently, setting `GIT_AUTHOR_NAME` and `GIT_AUTHOR_EMAIL` in the subprocess environment before launching the agent.

**Benefits:**
- Git history in the sandbox carries agent provenance
- `git log --author="claude-code"` filters agent commits instantly
- `git blame` distinguishes agent lines from human lines
- Session reports can link to specific commits by agent identity
- Near-zero implementation cost

**What this does NOT require:**
- SSH keys (irrelevant for local sandbox operations)
- Service accounts (enterprise-scale, not local-dev-scale)
- Certificate infrastructure (OWASP ASI03 suggests this; it is overkill here)
- Identity management systems (Principal Skinner's full proposal)

---

## 5. What Principal Skinner Over-Engineers

| Proposal | Rationale | Verdict for Rein |
|----------|-----------|-------------------|
| Per-agent SSH keys | Cryptographic identity proof | Over-engineering. Rein runs agents locally in sandboxes. SSH keys solve a remote-access authentication problem rein does not have. |
| Per-agent service accounts | Credential isolation | Over-engineering for local dev. Relevant for CI/CD pipelines or multi-tenant enterprise deployments. |
| Immutable audit logs | Tamper-proof forensics | Rein's JSON reports are already append-only per session. "Immutable" implies a separate log store (S3, append-only DB) — unnecessary for local development. |
| Agent IDs across sessions | Cross-session behavioral tracking | Partially useful. Rein could assign stable agent IDs, but cross-session tracking is a different feature (behavioral profiling) that requires a persistence layer rein does not have. |

The "Anthropic Attack" article's call for logging "every decision, every tool call, every observation" with agent identity is enterprise-grade observability. For a solo developer running tasks locally, session-level reports with git attribution cover 90% of the forensic value at 5% of the implementation cost.

---

## 6. OWASP ASI03 Alignment

| ASI03 Mitigation | Principal Skinner | Rein (Current) | Rein (With Git Identity) |
|-----------------|-------------------|-------------------|---------------------------|
| Isolated identities per agent | Proposed (SSH keys, service accounts) | Partial (report-level only) | Strong (report + git level) |
| Short-lived credentials | Proposed | N/A (no agent credentials) | N/A |
| Task-scoped permissions | Proposed | Yes (sandbox per task) | Yes |
| Credential rotation | Proposed | N/A | N/A |
| Audit trail | Proposed (immutable logs) | Yes (structured JSON reports) | Yes (reports + git history) |

Rein satisfies ASI03's intent through containment rather than credential management. An agent in a sandbox with no access to production credentials cannot abuse credentials it does not have. Principal Skinner's credential-isolation approach solves the same problem for a different deployment model (agents with direct infrastructure access).

---

## 7. The Ideal State

For rein, the ideal agent identity model is:

1. **Git author per session** — agent name, model, and effort encoded in commit metadata. Implementation cost: minimal (environment variables in subprocess).
2. **Structured reports linked to commits** — the session report references the commit hash(es) produced during the session. Already partially implemented via git diff capture.
3. **Stable agent identifiers** — a consistent naming scheme (`claude-code/opus/high`) so that git history and reports use the same vocabulary.

This is not Principal Skinner's full identity model. It is the subset that delivers forensic value for local development without enterprise infrastructure overhead.

---

## 8. Recommendation

**Do now:** Add agent-specific git author configuration per session. Rein controls the subprocess environment; setting `GIT_AUTHOR_NAME` and `GIT_AUTHOR_EMAIL` before agent launch is a one-line change per adapter. This closes the attribution gap at near-zero cost.

**Do not:** Build SSH key infrastructure, service account management, or cross-session identity tracking. These solve problems rein does not have in its current deployment model (local, single-operator, task-scoped sandboxes).

**Revisit when:** Rein supports CI/CD or multi-operator deployment, where credential isolation and cryptographic identity become necessary.

---

## Sources

- "Supervising Ralph: Why Every Wiggum Loop Needs a Principal Skinner." securetrajectories.substack.com, 2026.
- "The Anthropic Attack." securetrajectories.substack.com, 2026.
- OWASP. "Top 10 for Agentic Applications 2026." owasp.org — ASI03 (Identity & Privilege Abuse), ASI10 (Rogue Agents).
- Rein ARCHITECTURE.md, SESSIONS.md, TOKENS.md.
- research/ralph_wiggum_deep_dive/04_failure_modes.md — Section 4.2 (No Action Attribution).
- research/principal_skinner_deep_dive/00_synthesis.md.
