# Deep Dive Plan: Principal Skinner Rein Pattern

## Context for the Executing Agent

You are creating a deep dive research document for the **rein** project — a context-pressure-aware orchestrator for AI coding agents (Claude Code, Codex CLI, Gemini CLI). Rein monitors how agents consume their context windows, intervenes when quality is at risk (via green/yellow/red zone thresholds), and ensures work is persisted before degradation sets in. It uses subprocess isolation, real-time token monitoring, and structured evaluation with binary scoring.

Rein project has already completed deep dives on:
- **Goose** (Block's open-source agent framework) — in `research/goose_deep_dive/`
- **Stripe Minions** (Stripe's internal agent system) — in `research/stripe_deep_dive/`
- **RLMs** (Recursive Language Models) — in `research/rlm_deep_dive/`
- **Ralph Wiggum Loop** — in `research/ralph_wiggum_deep_dive/`

The **Ralph Wiggum Loop** is a viral agentic coding pattern that runs an AI coding agent in an infinite bash loop (`while :; do cat PROMPT.md | claude ; done`). Progress persists in files and git, not in the LLM's context window. It went viral in late 2025 and received an official Claude Code plugin from Anthropic.

The **Principal Skinner** pattern is a supervision and safety rein designed to wrap Ralph Wiggum Loops (and agentic coding loops in general). It was articulated in a Substack post at `securetrajectories.substack.com/p/ralph-wiggum-principal-skinner-agent-reliability`. The core argument: prompt-level safety instructions are probabilistic (the agent can ignore them), so safety must be enforced at the infrastructure level — deterministic tool-use control, behavioral circuit breakers, distinct agent identity, and adversarial simulation.

The Principal Skinner pattern is directly relevant to rein because it addresses the same problem rein solves: how do you safely supervise an autonomous coding agent? Rein uses subprocess isolation, zone-based intervention, and structured evaluation. Principal Skinner uses tool-use interception, circuit breakers, and pre-deployment adversarial testing. Understanding where these overlap and where they differ is the goal of this deep dive.

## Output Structure

Create the following files in `research/principal_skinner_deep_dive/`:

### 00_synthesis.md — Synthesis & Recommendations
- Executive summary (3-4 paragraphs)
- The Principal Skinner thesis: infrastructure-level safety vs prompt-level safety
- How rein already implements (or doesn't implement) each Principal Skinner mechanism
- Transferable patterns worth adopting in rein
- Tiered recommendations: Now / Next / Never
- Sources (consolidated)

### 01_safety_model.md — Safety Model & Philosophy
- The core argument: probabilistic vs deterministic safety
- OWASP Agentic Misalignment (ASI08) classification and how it applies
- The "overbaking" failure mode: definition, examples, detection
- "Sycophancy loops" — when the agent overrides safety constraints to satisfy the user
- How this relates to Rein's existing safety model (subprocess isolation, sandbox, zone kills)
- Whether rein has gaps that Principal Skinner addresses

### 02_tool_use_control.md — Deterministic Tool-Use Lanes
- How tool-use interception works (pre-execution hook? proxy? syscall filter?)
- Allowlisting vs blocklisting approaches
- Granularity: per-command? per-file-path? per-argument?
- How this compares to Rein's subprocess isolation model
- Whether rein should add tool-use interception or whether subprocess isolation is sufficient
- Practical examples: what commands get blocked, what gets allowed
- Relationship to Claude Code's permission model and hooks system

### 03_circuit_breakers.md — Behavioral Circuit Breakers
- What triggers a circuit breaker (dangerous command frequency? resource usage? time?)
- How circuit breakers differ from iteration caps
- Human-in-the-loop escalation: how it works, what the operator sees
- Comparison with Rein's zone-based intervention (yellow = graceful stop, red = kill)
- Whether rein needs circuit breakers beyond zone thresholds
- The distinction between "financial circuit breaker" (iteration cap) and "behavioral circuit breaker" (action monitoring)

### 04_agent_identity.md — Agent Identity & Attribution
- Unique SSH keys, service accounts, Agent IDs per agent
- How agent-authored commits are distinguished from human commits
- Audit trail design: what's logged, where, in what format
- How this compares to Rein's session reports (which already track agent identity per task)
- Whether rein needs stronger attribution (e.g., agent-specific git config per session)

### 05_adversarial_simulation.md — Pre-Deployment Adversarial Testing
- The concept: run thousands of simulated trajectories to find "toxic flows"
- What constitutes a toxic flow (infinite loops, destructive actions, data exfiltration)
- How simulation results inform policy (allowlists, circuit breaker thresholds)
- Whether this is practical for rein (cost, time, implementation complexity)
- Relationship to Rein's multi-run evaluation recommendation from the RLM deep dive
- Could Rein's evaluation framework be extended for adversarial simulation?

### 06_critical_analysis.md — Critical Analysis
- Is Principal Skinner a real implementation or a conceptual framework?
- What evidence exists for its effectiveness? (benchmarks, case studies, deployments)
- Does it over-engineer the safety problem for typical coding agent use cases?
- The tradeoff: safety overhead vs agent autonomy and speed
- What rein already does better than Principal Skinner
- What Principal Skinner does better than rein
- Gaps in both systems

## Research Sources (Start Here)

1. **Primary:** `https://securetrajectories.substack.com/p/ralph-wiggum-principal-skinner-agent-reliability` — The original Principal Skinner article
2. **Primary:** Search for any additional posts on `securetrajectories.substack.com` about agent safety, tool-use control, or circuit breakers
3. **Context:** Read `research/ralph_wiggum_deep_dive/00_synthesis.md` and `research/ralph_wiggum_deep_dive/04_failure_modes.md` in this repo for Ralph Loop failure modes that Principal Skinner addresses
4. **Context:** Read `ARCHITECTURE.md`, `SESSIONS.md`, `BRIEF.md` in this repo for Rein's existing safety model
5. **Background:** Search for OWASP Agentic Security guidelines, particularly ASI08 (Agentic Misalignment)
6. **Background:** Search for "agent safety rein" "tool use control" "agentic circuit breaker" for related work
7. **Background:** Read `research/06_agent_sandboxing_isolation.md` in this repo for existing sandboxing research

## Quality Criteria

- Every claim must cite a source (file path, URL, or document reference)
- Clearly distinguish between what Principal Skinner *proposes* and what it *implements* — if it's a conceptual framework with no code, say so
- Compare against Rein's existing capabilities in every section
- Be adversarial: identify over-engineering, impractical proposals, and missing evidence
- Assess whether each Principal Skinner mechanism is worth the implementation cost for rein
- Keep each document under 200 lines — concise and actionable
- Use the same tone and structure as existing deep dives in `research/ralph_wiggum_deep_dive/`
