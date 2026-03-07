# Proposal: OWASP Agentic Safety Mapping

**Target:** ARCHITECTURE.md (new section)
**Priority:** Now
**Effort:** Low

---

## Proposed Section: "Safety Model — OWASP Agentic Top 10 Mapping"

The following section is proposed for addition to ARCHITECTURE.md, after the "Future: Multi-Agent" section (or after the Ralph Ecosystem Positioning section if proposal 01 from ralph-orchestrator is adopted).

---

### Safety Model — OWASP Agentic Top 10 Mapping

The OWASP Top 10 for Agentic Applications (2026) is the industry-standard framework for reasoning about AI agent security risks. The harness's existing controls map to this framework as follows:

| OWASP Risk | ID | Harness Coverage | Mechanism | Gap |
|---|---|---|---|---|
| **Agent Goal Hijack** | ASI01 | Moderate | Operator-defined task prompts (not user-controlled). Sandbox isolation limits blast radius. | No input sanitization on task definitions. |
| **Tool Misuse** | ASI02 | Strong | Sandbox isolation (worktree/tempdir/copy). Agent cannot damage main working tree. Quality gate catches bad output. | No pre-execution tool-call inspection. Agent can misuse tools *within* sandbox. |
| **Identity & Privilege Abuse** | ASI03 | Partial | Each session reports agent name, model, effort. Subprocess runs with operator's permissions. | Agent uses operator's credentials. No per-agent credential scoping. See proposal [02](../proposals/02_agent_git_identity.md). |
| **Supply Chain Vulnerabilities** | ASI04 | Out of scope | The harness invokes agents as CLIs — it does not manage agent plugins, MCP servers, or tool registries. | Agent's tool ecosystem is the agent's responsibility. |
| **Unexpected Code Execution** | ASI05 | Strong | Sandbox containment. All agent execution is in an isolated directory. Post-execution validation catches unintended side effects. | No runtime code analysis within sandbox. |
| **Memory & Context Poisoning** | ASI06 | Strong | Fresh context per session (no conversation carryover). Seed files are operator-controlled, not agent-writeable across sessions. | LEARNINGS.md seed file (if adopted) creates a controlled cross-session memory channel. |
| **Insecure Inter-Agent Communication** | ASI07 | N/A (single-agent MVP) | No inter-agent communication exists. Review agent is invoked by the harness, not by the implementation agent. | Will need addressing when multi-agent mode is built. |
| **Cascading Failures** | ASI08 | Strong | Zone-based kills (Green/Yellow/Red). Token budget caps. Max 2 CI rounds. Stagnation detection (planned). | No action-content circuit breakers. See proposal [04](../proposals/04_action_frequency_monitoring.md). |
| **Human-Agent Trust Exploitation** | ASI09 | Moderate | Structured reports with scores prevent blind trust. Review agent provides independent assessment. | Operator must read reports — no automated escalation on suspicious results. |
| **Rogue Agents** | ASI10 | Strong | Subprocess SIGTERM/SIGKILL kill switch. Sandbox containment. Quality gate rejects bad output. Sandbox discarded on failure. | No real-time behavioral monitoring during execution. |

**Overall assessment:** The harness has strong coverage for ASI02, ASI05, ASI06, ASI08, and ASI10 through its containment-and-evaluation model. The primary gaps are ASI03 (agent identity — addressed by proposal 02) and the absence of pre-execution action inspection (partially addressed by proposal 03, Claude Code hooks). ASI04 and ASI07 are out of scope for the current single-agent, local-first architecture.

The harness's safety philosophy is **containment + post-execution evaluation**, not **pre-execution interception**. This covers the same threat surface as the Principal Skinner model for local development use cases, at significantly lower implementation cost. For enterprise or multi-agent deployments, pre-execution controls (ASI02) and inter-agent authentication (ASI07) would need revisiting.

Source: [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)

---

## Rationale

The Principal Skinner deep dive identified that the harness has strong safety controls but no systematic mapping to an established framework. The OWASP Agentic Top 10 provides:

1. **A shared vocabulary** for discussing agent safety with external stakeholders (security reviewers, enterprise evaluators, open-source contributors).

2. **Systematic gap identification** — rather than discovering gaps reactively (e.g., "should we add tool-use interception?"), the mapping shows which categories are covered, which are partially covered, and which are out of scope.

3. **Positioning clarity** — the mapping makes explicit that the harness's containment model covers most OWASP categories without the overhead of pre-execution interception. This is a design *choice*, not a gap.

This is documentation work — no code changes, no new subsystems. The mapping can be updated as the harness evolves (e.g., when multi-agent mode adds ASI07 relevance).

## Source References

- [Principal Skinner Synthesis](../00_synthesis.md) — Section 4.1 (OWASP as safety checklist)
- [Safety Model](../01_safety_model.md) — Section 2 (OWASP ASI02/ASI08/ASI10 analysis)
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
- ARCHITECTURE.md — current safety mechanisms (subprocess isolation, zone kills)
