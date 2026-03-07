# Enrichment Proposals: Overview & Rationale

**March 2026**

Proposals for enriching rein design docs with safety patterns identified in the Principal Skinner deep dive. Each proposal is self-contained and targets a specific rein doc. No existing files are modified — these are proposed additions for review.

---

## Proposal Index

| # | Proposal | Target Doc | Priority | Source |
|---|----------|-----------|----------|--------|
| [01](01_owasp_safety_mapping.md) | OWASP Agentic Safety Mapping | ARCHITECTURE.md | Now | [synthesis](../00_synthesis.md) §4.1, [safety model](../01_safety_model.md) §2 |
| [02](02_agent_git_identity.md) | Agent Git Identity Per Session | SESSIONS.md | Now | [synthesis](../00_synthesis.md) §4.2, [agent identity](../04_agent_identity.md) §4 |
| [03](03_claude_code_hooks.md) | Claude Code Hooks for Action Signals | AGENTS.md | Now | [synthesis](../00_synthesis.md) §4.3, [tool-use control](../02_tool_use_control.md) §5 |
| [04](04_action_frequency_monitoring.md) | Action Frequency Monitoring | SESSIONS.md | Next | [synthesis](../00_synthesis.md) §4.4, [circuit breakers](../03_circuit_breakers.md) §4 |
| [05](05_adversarial_evaluation.md) | Multi-Run Adversarial Evaluation | QUALITY_GATE.md | Next | [synthesis](../00_synthesis.md) §5, [adversarial simulation](../05_adversarial_simulation.md) §5 |

---

## Decision Rationale

### Why These Five

Each proposal addresses a gap where Principal Skinner correctly identifies a problem and rein has a practical, cost-effective path to addressing it:

1. **OWASP Agentic Safety Mapping** — Rein has strong safety controls but no systematic mapping to an established framework. OWASP Agentic Top 10 provides the vocabulary. This is documentation work, not engineering.

2. **Agent Git Identity** — Principal Skinner's attribution argument is sound: "You can only debug what you can identify." Rein tracks identity in reports but not in git commits. A one-line `git config` per subprocess closes this gap at near-zero cost.

3. **Claude Code Hooks** — Principal Skinner proposes tool-use interception. Claude Code's hook system (PreToolUse, PostToolUse) already provides this — rein doesn't need to build its own. Investigating and documenting hook integration is the pragmatic path.

4. **Action Frequency Monitoring** — Rein's one real gap from the Principal Skinner analysis: it monitors *how much* context is consumed but not *what actions* the agent takes. Simple tool-call counting bridges this without a full policy engine.

5. **Multi-Run Adversarial Evaluation** — Principal Skinner's "thousands of trajectories" is impractical, but running the same task 5-20 times with prompt variation captures 80% of the value. This connects to the RLM deep dive's multi-run recommendation.

### What Was Rejected (and Why)

These Principal Skinner patterns were evaluated and rejected:

| Pattern | Why Rejected | Source |
|---------|-------------|--------|
| **Custom tool-use interception layer** | Claude Code hooks + sandbox isolation already cover this. Building a bespoke interception engine is over-engineering. | [tool-use control](../02_tool_use_control.md) §6 |
| **Real-time policy-as-code engine** | Cedar/OPA-style policy evaluation adds latency and complexity disproportionate to solo/small-team risk. Sondera demonstrates the approach; rein does not need to replicate it. | [tool-use control](../02_tool_use_control.md) §4 |
| **Per-agent SSH keys & service accounts** | Enterprise-grade identity infrastructure for local development is overhead without benefit. Git author config provides sufficient attribution. | [agent identity](../04_agent_identity.md) §5 |
| **Pre-deployment adversarial simulation** | Running thousands of trajectories per task is cost-prohibitive ($100s-$1000s). The quality gate with validation commands provides equivalent assurance. | [adversarial simulation](../05_adversarial_simulation.md) §4 |
| **Deterministic tool allowlisting** | Signature-based matching is an arms race (Sondera's known limitation). Sandbox containment is simpler and harder to bypass. | [critical analysis](../06_critical_analysis.md) §4 |

### Mapping to Synthesis Tiers

| Tier | Proposals |
|------|-----------|
| **Now** (low effort, no new infrastructure) | 01 (OWASP mapping), 02 (git identity), 03 (hooks) |
| **Next** (when evidence justifies) | 04 (action monitoring), 05 (adversarial evaluation) |
| **Never** | Custom interception layer, policy-as-code engine, SSH keys, pre-deployment simulation |

---

## Sources

- [Principal Skinner Deep Dive: Synthesis](../00_synthesis.md)
- [Principal Skinner Deep Dive: Safety Model](../01_safety_model.md)
- [Principal Skinner Deep Dive: Critical Analysis](../06_critical_analysis.md)
- [Ralph Orchestrator Proposals](../../ralph_orchestrator_deep_dive/proposals/00_overview.md) — pattern template
- Rein design docs: ARCHITECTURE.md, SESSIONS.md, AGENTS.md, QUALITY_GATE.md
