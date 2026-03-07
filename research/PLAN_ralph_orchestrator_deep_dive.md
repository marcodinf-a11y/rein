# Deep Dive Plan: ralph-orchestrator

## Context for the Executing Agent

You are creating a deep dive research document for the **rein** project — a context-pressure-aware orchestrator for AI coding agents (Claude Code, Codex CLI, Gemini CLI). The rein monitors how agents consume their context windows, intervenes when quality is at risk (via green/yellow/red zone thresholds), and ensures work is persisted before degradation sets in. It uses subprocess isolation, real-time token monitoring, and structured evaluation with binary scoring.

The rein project has already completed deep dives on:
- **Goose** (Block's open-source agent framework) — in `research/goose_deep_dive/`
- **Stripe Minions** (Stripe's internal agent system) — in `research/stripe_deep_dive/`
- **RLMs** (Recursive Language Models) — in `research/rlm_deep_dive/`
- **Ralph Wiggum Loop** — in `research/ralph_wiggum_deep_dive/`

The Ralph Wiggum Loop is a viral agentic coding pattern created by Geoffrey Huntley in late 2025. It runs an AI coding agent in an infinite bash loop (`while :; do cat PROMPT.md | claude ; done`) where progress persists in files and git history rather than the LLM's context window. Each iteration gets fresh context, reads specs from disk, picks one task, implements it, commits, and exits. The loop restarts the process.

**ralph-orchestrator** (`github.com/mikeyobrien/ralph-orchestrator`) is a community-built enhancement of the Ralph Wiggum Loop that adds context window management and token tracking — moving the bare while loop toward structured orchestration. It is the closest community implementation to the rein's approach and represents the natural evolution of Ralph toward production-grade tooling.

This deep dive's purpose is to understand ralph-orchestrator's architecture, compare it to the rein, identify transferable patterns, and assess its maturity.

## Output Structure

Create the following files in `research/ralph_orchestrator_deep_dive/`:

### 00_synthesis.md — Synthesis & Recommendations
- Executive summary (3-4 paragraphs)
- What ralph-orchestrator adds over bare Ralph (table comparing bare Ralph vs ralph-orchestrator vs rein)
- Transferable patterns worth adopting in rein
- Patterns to avoid and why
- Tiered recommendations: Now / Next / Never
- Sources (consolidated)
- Links to all deep dive documents

### 01_architecture.md — Architecture & Design
- Overall system architecture (with ASCII diagram if possible)
- How it wraps the agent process (subprocess? in-process? hook?)
- State management: how progress is tracked between iterations
- Configuration model: what's configurable, what's hardcoded
- How it differs from the bare `while :; do ... ; done` loop
- Dependency footprint and language/runtime requirements

### 02_context_management.md — Context Window Management
- How it tracks token usage (real-time streaming? post-hoc? estimated?)
- Token thresholds and what actions they trigger
- Comparison with Rein's green/yellow/red zone model
- How it handles context rotation (process restart vs session continuation)
- Whether it supports the "deterministic allocation" pattern (re-loading specs each iteration)
- Compaction awareness: does it detect or prevent compaction events?

### 03_iteration_control.md — Iteration Control & Convergence
- How it decides when to continue vs stop
- Completion detection mechanism (completion promise? test results? custom?)
- Stagnation detection: does it detect repeated failures?
- Oscillation detection: does it detect fix-A-break-B cycles?
- Iteration limits and cost caps
- Comparison with Rein's quality gate and token budget

### 04_evaluation_reporting.md — Evaluation & Reporting
- What data it captures per iteration (tokens? diffs? test results? timing?)
- Report format and structure
- Whether it supports normalized comparison across runs
- Whether it produces machine-readable output (JSON, NDJSON)
- Comparison with Rein's structured reports and binary scoring

### 05_critical_analysis.md — Critical Analysis
- What ralph-orchestrator gets right that bare Ralph doesn't
- What it still gets wrong compared to rein
- Gaps: brownfield support, multi-agent, security, cost control
- Maturity assessment: production-ready? experimental? proof-of-concept?
- Community adoption and maintenance health (commits, contributors, issues)
- The honest question: does rein need anything from ralph-orchestrator, or has it already surpassed it?

## Research Sources (Start Here)

1. **Primary:** `https://github.com/mikeyobrien/ralph-orchestrator` — Read the README, source code, configuration files, and any documentation
2. **Context:** `https://medium.com/@sponge-theory.ai/ralph-orchestrator-solving-the-context-window-crisis-in-ai-powered-development-d91cee615656` — Medium article on the orchestrator
3. **Background:** Read `research/ralph_wiggum_deep_dive/00_synthesis.md` in this repo for the Ralph Loop baseline
4. **Background:** Read `ARCHITECTURE.md`, `TOKENS.md`, `SESSIONS.md` in this repo for rein design
5. **Web search:** Search for "ralph-orchestrator" and "mikeyobrien ralph" for any additional articles, discussions, or forks

## Quality Criteria

- Every claim must cite a source (file path, URL, or document reference)
- Comparison tables must include all three systems (bare Ralph, ralph-orchestrator, rein) where relevant
- Do not speculate about features — if the source code doesn't show it, say "not implemented"
- Be adversarial: identify what ralph-orchestrator does poorly, not just what it does well
- Keep each document under 200 lines — concise and actionable
- Use the same tone and structure as existing deep dives in `research/ralph_wiggum_deep_dive/`
