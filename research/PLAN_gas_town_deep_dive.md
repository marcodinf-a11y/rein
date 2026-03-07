# Deep Dive Plan: Gas Town Multi-Agent Orchestration

## Context for the Executing Agent

You are creating a deep dive research document for the **rein** project — a context-pressure-aware orchestrator for AI coding agents (Claude Code, Codex CLI, Gemini CLI). Rein monitors how agents consume their context windows, intervenes when quality is at risk (via green/yellow/red zone thresholds), and ensures work is persisted before degradation sets in. It uses subprocess isolation, real-time token monitoring, and structured evaluation with binary scoring.

Rein's current scope is single-agent task dispatch (one task → one agent → one session). Multi-agent support is planned for **workflow composition** — using different agents and models for different roles (e.g., plan with Claude/Opus, implement with Gemini/Flash, review with Claude/Sonnet). The architecture supports this from day one (agent adapter protocol, normalized token model, per-task agent/model/effort fields) but the MVP does not implement parallel dispatch.

Rein project has already completed deep dives on:
- **Goose** (Block's open-source agent framework) — in `research/goose_deep_dive/`
- **Stripe Minions** (Stripe's internal agent system) — in `research/stripe_deep_dive/`
- **RLMs** (Recursive Language Models) — in `research/rlm_deep_dive/`
- **Ralph Wiggum Loop** — in `research/ralph_wiggum_deep_dive/`

The **Ralph Wiggum Loop** is a viral agentic coding pattern that runs a single AI coding agent in an infinite bash loop. Progress persists in files and git. It went viral in late 2025.

**Gas Town** is Geoffrey Huntley's emerging infrastructure project for coordinating **multiple autonomous Ralph Loops into self-evolving ecosystems**. Where Ralph is a single agent in a single loop, Gas Town orchestrates ten or more agents concurrently, each on different tasks, with an orchestration layer managing coordination, conflicts, and resource allocation. Huntley describes it as "a complete rethinking of infrastructure to manage the chaos."

Gas Town is directly relevant to rein because it represents the multi-agent evolution of the Ralph pattern — and Rein's planned multi-agent workflow composition addresses the same problem space. Understanding Gas Town's approach, its successes and failures, and how it compares to Rein's planned multi-agent design is critical for informing Rein's roadmap.

## Output Structure

Create the following files in `research/gas_town_deep_dive/`:

### 00_synthesis.md — Synthesis & Recommendations
- Executive summary (3-4 paragraphs)
- What Gas Town is and what problem it solves
- Comparison table: single Ralph Loop vs Gas Town vs rein (planned multi-agent)
- Transferable patterns for Rein's multi-agent design
- Tiered recommendations: Now / Next / Never
- Sources (consolidated)

### 01_architecture.md — Architecture & Multi-Agent Model
- Overall system architecture (with ASCII diagram if possible)
- How multiple agents are spawned and managed (process model)
- Task assignment: how work is distributed across agents
- Agent communication: do agents communicate with each other? Through what mechanism?
- Shared state: how agents coordinate on a shared codebase (file locking? branching? merge?)
- Configuration model: how the operator defines the multi-agent topology

### 02_coordination.md — Coordination & Conflict Resolution
- The core problem: multiple agents editing the same codebase concurrently
- How Gas Town handles merge conflicts (git-based? file-level locking? directory partitioning?)
- Dependency management: how it handles task A depending on task B's output
- Ordering guarantees: can the operator specify sequential dependencies?
- What happens when two agents modify the same file simultaneously
- Comparison with Rein's planned approach (sequential multi-task with per-task agent assignment)

### 03_resource_management.md — Resource & Cost Management
- How it tracks token usage across multiple concurrent agents
- Aggregate cost visibility: does the operator see total spend across all agents?
- Per-agent cost caps or budgets
- How it handles a runaway agent (one agent consuming disproportionate resources)
- Comparison with Rein's token budget model (per-task budgets, normalized accounting)

### 04_task_decomposition.md — Task Decomposition & Planning
- How tasks are decomposed for multi-agent execution
- Operator-driven vs model-driven decomposition
- Whether Gas Town supports hierarchical task structures (plan → subtasks → sub-subtasks)
- How task granularity is determined (too coarse = conflicts, too fine = overhead)
- Comparison with Rein's operator-defined task sequences

### 05_critical_analysis.md — Critical Analysis
- **Maturity:** Is Gas Town released? Open source? Documented? Or is it vaporware / in-progress?
- **Evidence:** Any demonstrated results? Benchmarks? Case studies? Or just blog posts?
- **The fundamental problem:** Is concurrent multi-agent coding on a shared codebase tractable? What evidence exists either way?
- **Comparison with existing multi-agent systems:** How does Gas Town compare to Stripe Minions (which uses a blueprint/minion pattern with isolated agent scopes), Goose's subagent system, or academic multi-agent coding research?
- **What rein can learn:** Even if Gas Town is immature, are there architectural insights worth adopting?
- **What to avoid:** Patterns that add complexity without demonstrated value
- **The honest question:** Should rein pursue concurrent multi-agent execution, or is sequential multi-task with different agents per task sufficient?

## Research Sources (Start Here)

1. **Primary:** `https://ghuntley.com/loop/` — Huntley's "everything is a ralph loop" post, which introduces Gas Town
2. **Primary:** Search for "Gas Town" + "Geoffrey Huntley" + "agent" for any dedicated posts, talks, or repositories
3. **Primary:** Search for "ghuntley gas town" on GitHub for any public repositories
4. **Primary:** Search Hacker News, Twitter/X, and developer forums for Gas Town discussions
5. **Context:** Read `research/ralph_wiggum_deep_dive/00_synthesis.md` in this repo for the Ralph Loop baseline
6. **Context:** Read `research/stripe_deep_dive/00_synthesis.md` in this repo for Stripe's multi-agent approach (blueprint/minion pattern)
7. **Context:** Read `research/goose_deep_dive/03_recipes_subagents.md` for Goose's subagent model
8. **Context:** Read `ARCHITECTURE.md` and `BRIEF.md` in this repo for Rein's planned multi-agent support
9. **Context:** Read `research/09_multi_agent_patterns.md` for existing multi-agent research
10. **Background:** Search for "multi-agent coding" "concurrent AI agents" "shared codebase" for related academic and industry work

## Important Warning

Gas Town may be very early-stage, partially announced, or even vaporware. Huntley mentioned it in his "everything is a ralph loop" post as an emerging project. It may not have a public repository, documentation, or any implementation beyond a concept.

**If Gas Town has no public implementation or documentation:**
- Document what is known from blog posts and discussions
- Focus the deep dive on the *problem* Gas Town claims to solve (multi-agent coordination on shared codebases) rather than Gas Town's specific solution
- Survey how other systems solve this problem (Stripe Minions, Goose subagents, academic work)
- Recommend whether rein should pursue concurrent multi-agent execution based on the evidence from *all* sources, not just Gas Town
- Be explicit that Gas Town is conceptual/pre-release and adjust the analysis accordingly

## Quality Criteria

- Every claim must cite a source (file path, URL, or document reference)
- Clearly distinguish between what Gas Town *has demonstrated* and what it *claims to plan*
- If Gas Town is vaporware, say so explicitly and pivot the analysis to the problem domain
- Compare against both Rein's current design AND Rein's planned multi-agent support
- Include comparison with Stripe Minions and Goose subagents — these are Rein's existing reference points for multi-agent patterns
- Be adversarial: concurrent multi-agent coding may be a bad idea. Present the evidence for and against.
- Keep each document under 200 lines — concise and actionable
- Use the same tone and structure as existing deep dives in `research/ralph_wiggum_deep_dive/`
