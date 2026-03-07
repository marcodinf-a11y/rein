# Stripe Minions: Blueprint Pattern -- Deep Structural Analysis

## Executive Summary

The Blueprint is Stripe's core orchestration pattern for its Minions agentic CI/CD system. It defines a workflow that interleaves **deterministic code nodes** (git operations, linting, CI triggers) with **open-ended agent loops** (code generation, error diagnosis, fix application). The design philosophy is explicit: "The LLM only gets called when you actually need creativity or judgment. Everything else is just code." This report analyzes what is confirmed from primary sources, what can be reasonably inferred, and what remains unknown about the Blueprint pattern.

---

## Research Question 1: Deterministic Node vs. Open-Ended Agent Loop

### Confirmed (Source: Stripe Blog Posts Part 1 & Part 2; multiple third-party analyses)

Stripe explicitly describes Blueprints as "orchestration flows that alternate between fixed, deterministic code nodes and open-ended agent loops." The two node types are:

**Deterministic nodes** -- hardcoded pipeline steps that execute identically every time:
- Push to git
- Run the linter (a local executable that runs selected lints on each git push in under five seconds using heuristics)
- Trigger CI (selectively running tests from Stripe's battery of 3+ million tests)
- Apply autofixes (many tests have autofixes that are automatically applied)
- Git commit operations

**Agent loop nodes** -- LLM-driven steps where the model reasons and makes decisions:
- Implement the task (write the code)
- Fix CI failures (when no autofix exists, the failure goes back to the minion)
- Diagnose and resolve linter errors

The boundary rule is straightforward: **if the step can be expressed as deterministic code with predictable inputs and outputs, it is a deterministic node. If the step requires judgment, creativity, or reasoning about ambiguous inputs, it is an agent loop node.**

### Inferred (with reasoning)

The decision about which parts are deterministic and which are agentic is **made at Blueprint authoring time, not at runtime**. The blog posts describe Blueprints as a "coding framework" -- implying they are authored artifacts, not dynamically generated plans. This means a human engineer (or a team) decides the structure of each Blueprint in advance. The LLM does not decide which steps to take; it operates within the steps assigned to it.

This is consistent with Stripe's stated philosophy of "putting LLMs into contained boxes" -- the boxes are designed by humans, and the LLM fills them. The system "runs the model, not the other way around."

### Unknown

- Whether a single agent loop node can contain sub-loops (e.g., an inner retry cycle for a specific file) or whether retries are always managed at the Blueprint level.
- The exact mechanism by which an agent loop node signals completion -- whether it returns a structured output that the Blueprint validates, or whether the Blueprint simply reads the filesystem state.
- Whether there are hybrid nodes that start deterministic but can escalate to agentic (e.g., "try the autofix first; if it fails, hand to agent").

---

## Research Question 2: How Blueprints Are Defined

### Confirmed (Source: Stripe Blog Posts)

Stripe calls Blueprints "a coding framework that combines deterministic workflows with agent-like flexibility." The term "coding framework" strongly implies that Blueprints are **defined in code**, not in YAML/JSON config or via a GUI.

Minions run on a fork of Block's open-source Goose agent, "customized to interleave agent loops with deterministic code for git operations, linters, and testing." This confirms that Blueprints are implemented within the Goose fork's codebase.

### Inferred (with reasoning)

**Blueprints are most likely Rust or Python code** within Stripe's Goose fork. Goose itself is a Rust-based agent framework with the following crate structure:
- `goose` -- core agent logic
- `goose-cli` -- command-line interface
- `goose-server` -- backend for desktop app
- `goose-mcp` -- MCP server implementations

Goose's native orchestration primitive is the **recipe** system, which supports:
- Sequential execution of subrecipes (default for different subrecipes)
- Parallel execution of subrecipe instances (default for same subrecipe with different parameters)
- A `sequential_when_repeated` flag for configuration
- Subrecipes registered with a name, file path, and optional parameter values

It is highly likely that Stripe's Blueprint pattern is **built on top of or inspired by Goose's recipe system**, extended with deterministic node types that bypass the LLM entirely. The authoring experience is likely:

1. A Blueprint is a code module (Rust struct or function) that defines an ordered sequence of nodes.
2. Each node is either a `DeterministicNode` (runs a shell command, git operation, or API call) or an `AgentNode` (invokes the Goose agent loop with a specific prompt and tool subset).
3. Blueprints are registered in a catalog, keyed by task type.

**The authoring experience is code-first, not config-first.** This is consistent with Stripe's engineering culture and with Goose's architecture (which is explicitly "code-first" rather than config-driven).

### Unknown

- The exact API surface of the Blueprint framework -- whether nodes are defined declaratively (e.g., a list of node specs) or imperatively (e.g., sequential function calls with control flow).
- Whether Blueprints support parameterization (e.g., a generic "fix lint errors" Blueprint that accepts different lint configurations).
- Whether there is a Blueprint editor, test harness, or simulation mode for authors.
- The number of distinct Blueprints Stripe maintains (tens? hundreds?).

---

## Research Question 3: Branching Logic

### Confirmed (Source: Stripe Blog Posts)

The blog posts describe a specific branching pattern in the CI loop:

1. Local lint runs on each git push (deterministic, < 5 seconds).
2. If lint passes, CI selectively runs tests.
3. Many tests have autofixes that are **automatically applied** (deterministic branch: autofix exists -> apply it).
4. If no autofix exists, the failure goes **back to the minion** (agentic branch: no autofix -> agent fixes it).
5. At most **two CI rounds** to balance speed, cost, and diminishing returns.

This reveals a clear branching model:
- **Deterministic branching**: "if autofix exists, apply it" -- no LLM involved.
- **Agentic branching**: "if no autofix exists, feed errors to agent for fix" -- LLM reasons about the failure.
- **Hard cutoff branching**: "if 2 CI rounds exhausted, terminate" -- deterministic limit.

### Inferred (with reasoning)

The branching logic is **encoded in the Blueprint's control flow**, not decided by the LLM at runtime. The "2 CI rounds max" constraint is a Blueprint-level parameter, not something the agent negotiates. This means the Blueprint has conditional logic:

```
pseudocode:
for round in 1..MAX_CI_ROUNDS:
    deterministic: run_lint()
    if lint_fails:
        if autofix_available:
            deterministic: apply_autofix()
        else:
            agent: fix_lint_errors(errors)
    deterministic: run_ci_tests()
    if tests_pass:
        break
    else:
        if autofix_available:
            deterministic: apply_autofix()
        else:
            agent: fix_test_failures(failures)
terminate_with: create_pull_request()
```

This is a sequential loop with conditional branching -- not a DAG. The branching is simple: check a condition, choose a deterministic or agentic path. The loop bound is hard-coded.

### Unknown

- Whether Blueprints support more complex branching (e.g., "if the agent's fix introduces new lint errors, restart the lint cycle").
- Whether the agent receives the full CI output or a filtered/summarized version.
- Whether there are task-type-specific branching rules (e.g., migration Blueprints might have different branch logic than bug-fix Blueprints).

---

## Research Question 4: Execution Model

### Confirmed (Source: Stripe Blog Posts; Goose documentation)

The overall Minions execution flow is **sequential within a single task**:

```
Slack invocation -> isolated sandbox (devbox) -> MCP context assembly -> Blueprint execution -> CI loop -> human review -> merge
```

Within the Blueprint, execution is **sequential with bounded loops**. There is no evidence of parallel node execution within a single Blueprint run. The "one-shot" design philosophy reinforces this: a single minion handles a single task from start to finish, without forking into parallel sub-agents.

**Goose's underlying architecture** supports both sequential and parallel execution:
- Different subrecipes run sequentially by default
- Same subrecipe with different parameters runs in parallel by default
- Real-time progress dashboard for parallel execution

### Inferred (with reasoning)

The execution model is best described as a **sequential pipeline with bounded retry loops** rather than a DAG. Evidence:

1. The "one-shot" framing implies a linear flow, not a graph with multiple paths.
2. The "max 2 CI rounds" is a loop bound, not a fan-out.
3. There is no mention of parallel node execution or DAG scheduling in any source.
4. The devbox isolation model (one devbox per task) reinforces single-threaded execution per task.

However, **across tasks**, Stripe runs many minions in parallel. Each minion gets its own devbox, its own Blueprint execution, and its own CI pipeline. The parallelism is at the task level, not the node level.

This makes the execution model:
- **Intra-task**: Sequential pipeline with conditional branches and bounded loops
- **Inter-task**: Massively parallel (1,000+ PRs/week implies hundreds of concurrent minion runs)

### Unknown

- Whether any Blueprint nodes internally parallelize (e.g., running lints on multiple files concurrently).
- Whether there is a global scheduler that manages minion dispatch, queuing, and resource allocation across devboxes.
- The maximum number of concurrent minion runs.

---

## Research Question 5: State Flow Between Nodes

### Confirmed (Source: Stripe Blog Posts; Goose architecture)

**Context assembly happens before the Blueprint executes.** The orchestrator performs deterministic prefetching:
- Scans the prompt for links and keywords
- Pulls Jira tickets and relevant documentation
- Searches code via Sourcegraph using MCP
- Curates a surgical subset of approximately **15 relevant tools** from the 400+ Toolshed

This prefetched context is assembled into the agent's initial prompt. The agent does not discover tools dynamically -- they are pre-selected.

**Between nodes within a Blueprint:**
- The primary state medium is the **filesystem** (the devbox). A deterministic node runs a linter and produces output files/stdout. An agent node reads those outputs and modifies source files.
- Git operations serve as **checkpoints** -- deterministic git commit nodes capture the agent's work before the next phase.

### Inferred (with reasoning)

State flow between nodes likely works as follows:

1. **Deterministic -> Agent**: The deterministic node produces structured output (lint errors, test failures, CI logs). This output is injected into the agent's prompt as additional context. The agent also has filesystem access to see the current state of the code.

2. **Agent -> Deterministic**: The agent modifies files on the devbox filesystem. The deterministic node then operates on those files (e.g., `git add`, `git commit`, run linter).

3. **Shared state**: The devbox filesystem is the shared state store. There is no separate state database or message queue between nodes.

4. **Context window management**: Each agent node likely gets a **fresh context window** assembled from:
   - The original task description
   - The prefetched MCP context (documentation, code references)
   - The outputs of the preceding deterministic node (errors, logs)
   - The current file state on the devbox

This is consistent with the "one-shot" philosophy -- each agent invocation gets a complete, self-contained context payload rather than building on a growing conversation history.

### Unknown

- Whether agent nodes in later Blueprint stages (e.g., CI fix) receive the full original context or a trimmed version.
- Whether there is explicit state serialization between nodes (e.g., a JSON object passed from node to node) or only implicit state via the filesystem.
- How large the context payload is for a typical agent node invocation.
- Whether Stripe uses context window management techniques like compaction or summarization for multi-round CI fixes.

---

## Research Question 6: Partial Completion Handling

### Confirmed (Source: Stripe Blog Posts)

The primary constraint on partial completion is the **2 CI round limit**. If a minion cannot produce a passing PR within 2 CI rounds, the task terminates. The blog posts describe this as a deliberate balance of "speed, cost, and diminishing returns."

The "one-shot" design philosophy means that partial completion within a single agent invocation is less of a concern -- the agent either produces the full output or it does not. There is no multi-step agent conversation that can be interrupted mid-way.

### Inferred (with reasoning)

**Partial completion scenarios and likely handling:**

1. **Agent produces code but lint fails (round 1)**: The Blueprint loops -- agent gets another chance with lint errors as context. This is the normal flow, not partial completion.

2. **Agent produces code but tests fail after 2 rounds**: The task terminates. The PR is likely either:
   - Not created (task marked as failed)
   - Created in draft/WIP state for human triage
   - Logged with failure diagnostics for the requesting engineer

3. **Agent hits token/time limit mid-generation**: Since minions are "one-shot," the agent invocation either completes or fails entirely. There is no concept of "3 of 5 subtasks done" because each agent node handles a single coherent task (implement the feature, fix the errors).

4. **Infrastructure failure (devbox crash, network issue)**: The devbox is ephemeral and isolated. A crash likely means the task is retried from scratch on a new devbox.

The "one-shot" constraint is actually a **design decision that eliminates the partial completion problem**. By scoping each minion to a single, well-defined task (not a multi-part project), the system avoids the scenario where an agent completes 3 of 5 subtasks. If a task is too large for one-shot execution, the solution is to decompose it into smaller tasks before assigning to minions, not to handle partial completion gracefully.

### Unknown

- What happens to failed tasks after the 2-round limit -- do they go back to a human queue, get retried with a different model, or get decomposed differently?
- Whether there is a success rate metric (e.g., "X% of assigned tasks produce a merged PR").
- Whether Stripe has a task decomposition step that prevents over-large tasks from reaching minions.
- How token budget limits interact with the Blueprint execution -- whether there is a per-node token budget or only a per-task budget.

---

## Research Question 7: Blueprint-to-Task Relationship

### Confirmed (Source: Stripe Blog Posts; SitePoint analysis)

"Minion task definitions function as strict contracts between the orchestration layer and the LLM. Each definition specifies the task type, required input fields, expected output format, constraints the model must respect, and criteria for evaluating success."

The task definition is separate from the Blueprint. The task is "what to do"; the Blueprint is "how to orchestrate doing it."

### Inferred (with reasoning)

The relationship is most likely **many tasks to one Blueprint per task type**:

1. A Blueprint is a reusable orchestration template for a **category** of tasks (e.g., "fix lint errors," "add type annotations," "implement API endpoint," "update dependency").
2. Multiple specific tasks (Jira tickets, Slack requests) are dispatched to the same Blueprint based on their task type.
3. The task definition provides the **parameters** that customize the Blueprint execution (which files to modify, what the desired behavior is, which tests to run).

This is analogous to a CI pipeline template -- you have one pipeline definition for "run tests on PR," but many PRs flow through it with different parameters.

Evidence supporting this:
- The blog mentions specific categories of work that minions handle, implying task categorization.
- The "Toolshed" serves curated subsets of tools per task, which would be configured at the Blueprint level.
- Stripe's scale (1,000+ PRs/week) requires reusable patterns, not bespoke orchestration per task.

The task routing likely works as follows:
1. Developer posts task in Slack (or task is auto-generated from a ticket).
2. A classifier (possibly LLM-based, possibly rule-based) determines the task type.
3. The appropriate Blueprint is selected based on task type.
4. The Blueprint is parameterized with the specific task's details.
5. The Blueprint executes in a fresh devbox.

### Unknown

- The exact number of Blueprint types Stripe maintains.
- Whether Blueprints can be composed (e.g., a "migration Blueprint" that internally calls a "fix type errors sub-Blueprint").
- Whether new Blueprint types are frequently created or whether the set is stable.
- Whether the task-to-Blueprint routing is manual, rule-based, or LLM-based.
- Whether a single task can trigger multiple Blueprint executions (e.g., one task decomposed into multiple minion runs).

---

## Goose Architecture: The Foundation Stripe Built On

### Confirmed (Source: Goose GitHub, block.github.io documentation)

Goose is an open-source AI agent framework built in Rust by Block, Inc. Key architectural features:

**Crate structure:**
- `goose` -- core agent logic (agent loop, provider abstraction)
- `goose-cli` -- command-line interface
- `goose-server` -- backend server (supports multiple concurrent sessions)
- `goose-mcp` -- MCP server implementations
- `goose-bench` -- benchmarking
- `goose-test` -- test utilities

**Agent loop:** "Perceive -> Plan -> Execute -> Verify" closed loop. The engine interprets task intent, decomposes it into steps, invokes tools, and retries or adjusts strategy based on feedback.

**Extension system:** Built entirely on MCP. Extensions are MCP servers that provide tools. Over 3,000 MCP servers available in the ecosystem.

**Recipe system:** Reusable orchestration templates.
- Subrecipes for specialized tasks with name, path, and parameter values
- Parallel execution of subrecipe instances (default for same subrecipe with different params)
- Sequential execution of different subrecipes (default)
- `sequential_when_repeated` flag for configuration

**Session management:** Per-session agent instances; enabling/disabling extensions in one session does not affect others.

**Provider agnostic:** Supports any LLM provider; sends requests with available tool lists and processes tool call responses.

### What Stripe Likely Changed in Their Fork

Based on the gap between Goose's architecture and Stripe's described Blueprint pattern:

1. **Added the Blueprint orchestration layer** on top of Goose's recipe system -- interleaving deterministic nodes with agent loops rather than running pure agent loops.
2. **Built the Toolshed** as a centralized MCP server hosting 400+ internal tools, with a selection/curation layer that filters to ~15 tools per task.
3. **Added deterministic prefetching** -- the context assembly pipeline that scans prompts, pulls tickets, and searches code before the agent activates.
4. **Integrated with Stripe's devbox infrastructure** for isolated execution environments.
5. **Added CI integration** -- the lint/test/autofix loop with the 2-round cap.
6. **Added Slack invocation** as the entry point.

---

## Comparison: Blueprint Pattern vs. Industry Orchestration Patterns

| Dimension | Stripe Blueprint | LangGraph | CrewAI Flows | Google ADK Workflows |
|---|---|---|---|---|
| **Execution model** | Sequential pipeline + bounded loops | Stateful graph (DAG) | Event-driven pipelines | Sequential/Parallel/Loop agents |
| **Node types** | Deterministic + Agent | All LLM-driven (with tool calls) | Crews (autonomous) + code steps | LlmAgent + Workflow agents |
| **State passing** | Filesystem + prompt injection | Shared state object with checkpoints | Task outputs + shared memory | Session state with magic prefixes |
| **Branching** | Code-level conditionals | Graph edges with conditions | Event-driven routing | LLM-driven transfer |
| **Parallelism** | Inter-task only | Intra-graph parallel branches | Crew-level parallelism | Parallel agent node |
| **Retry/loop** | Max 2 CI rounds (hard cap) | Configurable cycles | Task retry via Flows | Loop agent with termination |
| **Human-in-loop** | PR review gate (end only) | Native HITL at any node | HITL for Flows | Native HITL |

The Blueprint pattern is notably **simpler** than most orchestration frameworks. It trades flexibility for reliability -- by constraining the LLM to specific "boxes" within a deterministic pipeline, it avoids the failure modes of complex multi-agent graphs.

---

## Key Architectural Insights for Rein

1. **Deterministic wrapping is the key pattern.** The highest-value insight from the Blueprint pattern is not the agent loop itself, but the deterministic nodes that surround it. Linting, testing, git operations, and CI triggers do not need LLM involvement. Rein should identify which steps in its pipeline can be deterministic.

2. **Tool curation, not tool proliferation.** Stripe's 400+ tools are available but only ~15 are exposed per task. This is a critical context window optimization. Rein should consider per-task tool filtering.

3. **One-shot eliminates partial completion.** By scoping tasks tightly enough that they can be completed in a single agent invocation (plus bounded retries), Stripe avoids the entire class of "agent got stuck halfway" problems. Rein's context pressure monitoring is solving a problem that Stripe sidesteps by design.

4. **Bounded retries with hard caps.** The 2 CI round limit is a pragmatic engineering decision, not an optimization. Diminishing returns on retries are real -- rein should consider similar hard caps.

5. **Filesystem as state store.** Stripe does not use a complex state management system between nodes. The devbox filesystem is the shared state. This is simple and effective for coding tasks.

---

## What Remains Unknown -- Open Questions

1. **Blueprint internals**: The exact code structure, API, and authoring workflow for Blueprints is not public. We know they exist and roughly what they do, but not their implementation.

2. **Failure rates**: Stripe publishes throughput (1,000+ PRs/week) but not success rate. How many tasks are attempted vs. how many produce merged PRs?

3. **Context management**: Stripe's blog posts never mention context window pressure, compaction, or degradation. Either they avoid the problem (via one-shot + short tasks) or they have internal solutions they do not publish.

4. **Model choice**: Stripe does not specify which LLM powers minions. This is significant for reproducibility.

5. **Task decomposition**: How tasks are scoped to be "one-shot-able" is not described. There must be a pre-processing step (human or automated) that ensures tasks are small enough.

6. **Blueprint evolution**: How new Blueprints are created, tested, and deployed is not described. Is there a Blueprint development lifecycle?

7. **Cost per PR**: No cost data is published. At frontier model pricing with 1,000+ PRs/week, this could be substantial.

---

## Sources

### Primary Sources (Stripe Official)
- [Minions: Stripe's one-shot, end-to-end coding agents -- Part 1](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)
- [Minions: Stripe's one-shot, end-to-end coding agents -- Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [Steve Kaliski on X (Part 2 announcement)](https://x.com/stevekaliski/status/2024578928430764362)

### Goose Architecture Sources
- [Goose GitHub Repository](https://github.com/block/goose)
- [Goose Architecture Documentation](https://block.github.io/goose/docs/goose-architecture/)
- [Goose Architecture Overview](https://block.github.io/goose/docs/category/architecture-overview/)
- [Building Custom Extensions](https://block.github.io/goose/docs/tutorials/custom-extensions/)
- [Running Subrecipes in Parallel](https://block.github.io/goose/docs/tutorials/subrecipes-in-parallel/)
- [Subrecipes for Specialized Tasks](https://block.github.io/goose/docs/guides/recipes/subrecipes/)
- [Goose AGENTS.md](https://github.com/block/goose/blob/main/AGENTS.md)
- [Unify Agent Execution Discussion #4389](https://github.com/block/goose/discussions/4389)
- [Goose OSS Roadmap (Feb-Apr 2026) Discussion #6973](https://github.com/block/goose/discussions/6973)
- [Block Introduces Goose](https://block.xyz/inside/block-open-source-introduces-codename-goose)

### Third-Party Analysis
- [Deconstructing Stripe's 'Minions': One-Shot Agents at Scale -- SitePoint](https://www.sitepoint.com/stripe-minions-architecture-explained/)
- [Stripe's coding agents: the walls matter more than the model -- Anup Jadhav](https://www.anup.io/stripes-coding-agents-the-walls-matter-more-than-the-model/)
- [What Stripe's Minions Get Right About Coding Agents -- Mr. Phil Games](https://www.mrphilgames.com/blog/what-stripes-minions-get-right-about-coding-agents)
- [Stripe Minions Research -- Ry Walker](https://rywalker.com/research/stripe-minions)
- [The Emerging "Harness Engineering" Playbook -- ignorance.ai](https://www.ignorance.ai/p/the-emerging-harness-engineering)
- [Hacker News Discussion -- Part 1](https://news.ycombinator.com/item?id=47110495)
- [Hacker News Discussion -- Part 2](https://news.ycombinator.com/item?id=47086557)
