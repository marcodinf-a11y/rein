# Gas Town: Task Decomposition & Planning

**March 2026**

---

## 1. How Tasks Are Decomposed

Gas Town uses a two-stage decomposition model:

### Stage 1: Human → Mayor (Strategic Decomposition)
The human operator provides a high-level prompt to the Mayor. The Mayor then:
1. Analyzes the existing codebase
2. Identifies required components (database, API, frontend, tests, etc.)
3. Creates individual Beads for each component
4. Establishes dependency ordering between Beads
5. Groups Beads into Convoys for coordinated delivery

### Stage 2: Mayor → Polecats (Tactical Dispatch)
The Mayor dispatches unblocked Beads to available Polecats based on:
- Dependency satisfaction (predecessors completed)
- Agent availability (free worktree slots)
- Task characteristics (some tasks route to Crew for domain expertise)

**Example** (from Better Stack guide): A single prompt for "add JWT authentication to a TODO app" generates separate Beads for:
- Core authentication backend (database, user models, JWT endpoints)
- HTML forms (login/register views)
- Test coverage (auth_test.go)
- Dockerfile generation

Source: [Better Stack Guide](https://betterstack.com/community/guides/ai/gas-town-multi-agent/)

---

## 2. Operator-Driven vs Model-Driven Decomposition

Gas Town uses **model-driven decomposition** — the Mayor (an LLM) decides how to break down work. The operator provides the high-level goal and the Mayor makes all decomposition decisions.

**Strengths:**
- Reduces cognitive load on the operator (one prompt, many tasks)
- Mayor can analyze the codebase to inform decomposition
- Enables rapid iteration (change the prompt, get a new decomposition)

**Weaknesses:**
- Decomposition quality depends on the LLM's understanding of the codebase
- No formal validation that the decomposition covers all requirements
- Vague prompts produce vague decompositions → wasted work
- The operator cannot easily preview or modify the decomposition before dispatch

Gas Town users consistently report that **specification quality is the bottleneck**, not coding speed:
- "The limiting factor is how fast you can think, give direction, and validate" — [Embracing Enigmas](https://embracingenigmas.substack.com/p/exploring-gas-town)
- "Planning and design become the limiting factor. Humans must deeply consider what to build before prompting agents" — [Maggie Appleton](https://maggieappleton.com/gastown)

---

## 3. Hierarchical Task Structures

Gas Town supports hierarchical decomposition through the MEOW stack:

```
Formula (TOML spec)
  └→ Protomolecule (workflow template)
      └→ Molecule (instantiated workflow)
          └→ Bead (atomic task)
              └→ Wisp (lightweight sub-task)
```

**Molecules** are the key abstraction — directed acyclic graphs (DAGs) of Beads with:
- **Dependencies**: Bead B waits for Bead A
- **Gates**: Checkpoint conditions that must be met before proceeding
- **Loops**: Retry patterns for tasks that may need multiple attempts

**Formulas** provide reusability — a "feature development" Formula defines the standard decomposition pattern (design → implement → test → integrate) that can be instantiated for any feature.

Source: [Torq Reading List](https://reading.torqsoftware.com/notes/software/ai-ml/agentic-coding/2026-01-15-gas-town-multi-agent-orchestration-framework/)

---

## 4. Task Granularity

Gas Town faces the classic granularity trade-off:

| Too Coarse | Too Fine |
|------------|----------|
| Agents conflict on shared files | Coordination overhead exceeds execution time |
| Context windows fill before completion | Dependency chains block parallelism |
| Merge conflicts are complex | Agent cold-start costs dominate |

Gas Town's pragmatic answer: **one Bead per coherent unit of work** — typically one module, one feature, or one test suite. The Mayor makes this judgment based on codebase analysis.

**Observed failure mode:** When the Mayor creates Beads that are too interdependent (e.g., "build the API" and "build the API tests" running concurrently), the test agent may generate tests against a non-existent or partially complete API, producing throwaway work.

---

## 5. Comparison with Related Systems

| System | Decomposition | Who Decides | Granularity Control |
|--------|--------------|-------------|---------------------|
| **Gas Town** | Model-driven (Mayor LLM) | Mayor agent | LLM judgment |
| **Stripe Minions** | Blueprint-driven (human) | Human architect | Blueprint scoping |
| **Ralph Loop** | Manual (PROMPT.md) | Human operator | One task per iteration |
| **Ralph Orchestrator** | Manual (task list) | Human operator | Configurable iteration scope |
| **Goosetown** | Model-driven (Orchestrator) | Orchestrator agent | Research/build/review phases |
| **Rein (planned)** | Operator-defined sequence | Human operator | Per-task definition |

**Key distinction:** Gas Town and Goosetown trust the LLM to decompose. Stripe Minions and rein keep decomposition as a human responsibility. The evidence from Gas Town users suggests that **human-driven decomposition produces better results** because specification quality is the bottleneck — and LLM decomposition amplifies vague specifications rather than catching them.

Source: `research/stripe_deep_dive/00_synthesis.md`, `research/ralph_orchestrator_deep_dive/00_synthesis.md`

---

## 6. Implications for Rein

### Keep operator-defined task sequences
Gas Town's experience validates Rein's choice to keep task decomposition as a human responsibility. Model-driven decomposition is convenient but produces unreliable results when specifications are vague. Rein should not delegate decomposition to the LLM unless it also validates the decomposition quality.

### Add specification validation
Before dispatching a task, rein could validate that the task description is specific enough for agent execution. Gas Town's biggest lesson: vague prompts → wasted compute. A lightweight pre-dispatch check ("does this task have clear acceptance criteria?") would catch the most common failure mode.

### Consider Formula-like templates
Gas Town's Formula concept (reusable workflow templates for common operations) is worth adopting in a simpler form. Rein could offer predefined task sequences for common operations (feature development, bug fix, refactoring) that operators customize rather than build from scratch.

### Avoid deep hierarchies
Gas Town's 5-layer MEOW stack (Bead → Epic → Molecule → Protomolecule → Formula) is over-engineered for most use cases. Rein should keep task structures flat: a sequence of tasks with optional dependencies. Two levels (task group → task) is sufficient.
