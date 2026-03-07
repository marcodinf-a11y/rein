# RLM Benchmarks, Evaluation, and Cost Analysis

How RLM performance data maps onto Rein's evaluation model, budget system, and quality gate.

Sources: [RLM paper (arXiv:2512.24601)](https://arxiv.org/abs/2512.24601), [Reproduction paper (arXiv:2603.02615)](https://arxiv.org/abs/2603.02615), rein specs (TOKENS.md, QUALITY_GATE.md, 07_agent_evaluation_benchmarks.md).

---

## 1. Performance by Task Complexity

The RLM paper reports results across four benchmarks with GPT-5. The key insight is that performance gains correlate with task complexity class, not input length.

| Benchmark | GPT-5 RLM | GPT-5 Base | Improvement | Complexity Class |
|-----------|-----------|-----------|-------------|------------------|
| CodeQA | 62% ($0.11) | 24% ($0.13) | +158% | Linear |
| BrowseComp+ 1K | 91.33% ($0.99) | 0% (exceeds window) | N/A | Multi-hop |
| OOLONG | 56.50% ($0.43) | 44% ($0.14) | +28.4% | Linear |
| OOLONG-Pairs | 58% ($0.33) | 0.04% ($0.16) | +1450x | Quadratic |

Source: Table 1, arXiv:2512.24601 v2.

### Complexity Classification

The paper identifies three complexity classes that determine how RLM cost scales:

- **Constant-complexity (S-NIAH)**: Single fact retrieval from long context. RLM cost stays roughly constant regardless of input size because the model can grep/search directly. Base GPT-5 matches or beats RLM for contexts under 2^14 tokens -- the REPL overhead is not justified (arXiv:2512.24601, Section 5.1).

- **Linear (OOLONG, CodeQA)**: Tasks requiring a single pass over the full context -- reading comprehension, entity extraction, chronological ordering. Cost grows linearly with input. RLM improvement ranges from +28% to +158%.

- **Quadratic (OOLONG-Pairs)**: Tasks requiring comparison of all pairs of elements in the context. Cost grows quadratically. This is where RLMs produce the most dramatic gains (58% vs 0.04%) because base models cannot hold enough working memory to track pairwise comparisons.

### Crossover Point

The S-NIAH results establish a crossover: RLMs become worthwhile when **task complexity exceeds what attention can handle in a single pass**. For simple retrieval on contexts under ~16K tokens, base models are cheaper and equally accurate. The crossover sits roughly at the point where either (a) the input exceeds the model's context window, or (b) the task requires multiple passes over the input (comparison, aggregation, multi-hop reasoning).

For rein, this means RLM wrapping should be conditional, not default. A task classifier or complexity heuristic would need to gate whether the RLM scaffold is invoked. No such classifier exists in the RLM paper or reference implementation -- the user decides manually.

---

## 2. CodeQA Translation to Rein Tasks

CodeQA tests reading comprehension on 900K-token code repositories: the model is given a large codebase and asked factual questions about it (e.g., "What does function X return when Y is true?"). GPT-5 RLM scores 62% vs 24% for the base model (arXiv:2512.24601, Table 1).

### What CodeQA Measures

CodeQA is a **read-only Q&A benchmark**. The model must navigate a large codebase, find relevant code, and answer questions about it. It never modifies files, runs tests, or commits changes.

### What the Rein Requires

Rein tasks are **action-oriented**: write code, fix bugs, run validation commands, produce diffs. The evaluation is binary -- all `validation_commands` pass (score 1.0) or any fail (score 0.0). The agent must modify the filesystem, not just comprehend it.

### Predictive Value

CodeQA performance predicts one component of agentic coding: **codebase navigation and comprehension**. An agent that cannot understand a large codebase cannot fix bugs in it. But CodeQA says nothing about:

- **Code generation quality** -- writing correct patches is harder than answering questions about existing code
- **Tool use and iteration** -- rein agents run tests, read errors, and iterate; CodeQA is single-shot
- **Diff scope control** -- rein tracks diff size; CodeQA has no analog
- **Side effects** -- filesystem modifications, dependency installation, build steps

The 62% CodeQA score suggests RLMs can navigate large codebases significantly better than base models. Whether that navigation advantage translates to better bug fixes is an open empirical question. No benchmark currently tests "RLM agent modifies a 900K-token codebase and produces a passing patch."

---

## 3. Cost Distribution Analysis

### Median vs Tail Costs

The RLM paper reports that median RLM runs are cheaper than base models on long-context tasks. On CodeQA, the RLM median cost is $0.11 vs $0.13 for base GPT-5 -- the RLM is actually 15% cheaper because it avoids processing 900K tokens through attention (arXiv:2512.24601, Table 1).

However, the distribution has heavy tails. The paper acknowledges that 95th-percentile runs can be 3-5x more expensive due to redundant verification loops and excessive sub-calls (arXiv:2512.24601, Section 6, Limitation 2; Appendix B.3 shows a model verifying its answer 5+ times).

### Interaction with Rein's Budget Model

Rein uses a default 70K token budget with three thresholds (TOKENS.md):

| Status | Utilization | Total Tokens | Signal |
|--------|------------|--------------|--------|
| WITHIN | < 80% | < 56,000 | Normal |
| WARNING | 80-100% | 56,000-70,000 | Context rot risk |
| EXCEEDED | > 100% | > 70,000 | Context rot likely |

The budget tracks `input_tokens + output_tokens` as `total_tokens`. This creates a tension with RLM cost profiles:

1. **RLM sub-calls are invisible to rein budget.** Rein monitors the top-level agent's token usage. If the agent invokes an RLM wrapper that makes 10 sub-LM calls, those sub-calls are billed by the API provider but may not surface in Rein's `NormalizedTokenUsage` extraction. Rein would need to aggregate tokens across the full RLM call tree, not just the root call.

2. **The 70K budget is calibrated for single-pass agents.** RLMs on linear/quadratic tasks routinely consume more tokens by design. A CodeQA RLM run at $0.11 with GPT-5 pricing translates to roughly 30-50K tokens depending on the pricing model -- within budget. But an OOLONG run at $0.43 or a BrowseComp+ run at $0.99 would likely exceed 70K tokens.

3. **Tail costs can blow past EXCEEDED.** If 95th-percentile runs are 3-5x the median, a task budgeted at 70K tokens could see outlier runs consuming 200K+ tokens. Rein's EXCEEDED status would trigger, but only if the sub-call tokens are visible to the budget tracker.

### Can the Budget Catch Runaway Costs?

Only if rein can observe RLM sub-call token usage in real time. The current architecture monitors agent output streams (Claude Code, Codex CLI, Gemini CLI) for token usage events. An RLM-powered agent would need to either:

- Report aggregate token usage including sub-calls through the same stream interface
- Expose a callback or event for each sub-call completion
- Accept a token ceiling parameter that the RLM framework enforces internally

The `rlms` library includes an `RLMLogger` that tracks per-call token usage to JSONL. Rein could parse these logs post-completion, but that provides no real-time budget enforcement.

---

## 4. Reproduction Study Variance

The reproduction paper (arXiv:2603.02615, Wang, March 2026) tested RLMs with DeepSeek v3.2 and Kimi K2 on S-NIAH and OOLONG benchmarks.

### Key Findings

- **Confirmed** RLM gains on complex tasks (OOLONG aggregation, pairwise comparison)
- **Degradation on simple retrieval**: Depth-1 RLMs performed worse than vanilla LLMs on S-NIAH -- the REPL overhead adds noise without benefit
- **Deeper recursion backfires**: Depth-2 RLMs scored significantly worse than depth-1 (DeepSeek v3.2 dropped from 42.1% to 33.7%) due to compounding formatting errors
- **Extreme run-to-run variance**: Scores ranged from 0/6 to 6/6 across 30 identical runs on aggregation tasks

### What 0/6-to-6/6 Variance Means

On a given aggregation task, running the same RLM configuration 30 times produced scores spanning the full range from complete failure to perfect accuracy. This is not sampling noise at the margins -- it is fundamental unpredictability.

For a coding workflow, this means:

- **A single run is not informative.** An RLM-powered agent might produce a perfect patch or a completely wrong one, with no way to predict which outcome a given run will produce.
- **Retry strategies become essential.** Rein's `max_rounds = 2` retry mechanism (QUALITY_GATE.md) partially addresses this -- a failed first attempt gets a second shot with feedback. But if the underlying variance is due to RLM decomposition strategy (not fixable bugs), retry may not help.
- **Majority voting is one mitigation.** Running the same task N times and taking the majority result reduces variance at the cost of N-fold expense. Rein does not currently support this pattern.
- **Production reliability is low.** The reproduction paper concludes that "large-scale industrial deployment remains highly challenging" (arXiv:2603.02615). For a system targeting reliable, repeatable coding workflows, RLM variance is a significant concern.

---

## 5. Rein Evaluation Compatibility

### Scoring Model Mismatch

Rein uses binary scoring: all `validation_commands` pass (score 1.0) or any fail (score 0.0). RLM papers report accuracy percentages across benchmark suites (62% on CodeQA means 62% of questions answered correctly across many test cases).

These are compatible in principle -- rein score for a single task is binary, and accuracy across many tasks gives a percentage. A suite of 100 rein tasks would yield an accuracy metric comparable to benchmark scores.

### Quality Gate Gaps for RLM Agents

The current quality gate (QUALITY_GATE.md) evaluates: build, tests, lint, typecheck, diff_size, context_pressure, and review. None of these signals capture RLM-specific behavior.

**Proposed new signals for RLM-powered agents:**

| Signal | What It Measures | Why It Matters |
|--------|-----------------|----------------|
| `sub_call_count` | Number of `llm_query()` sub-calls made | Detects excessive sub-calling (redundant verification). High counts correlate with cost spikes. |
| `recursion_depth` | Maximum depth of nested RLM calls | Current implementations cap at depth 1. Deeper recursion degrades quality (arXiv:2603.02615). |
| `repl_error_rate` | Fraction of REPL code executions that produced Python exceptions | High error rates indicate the model is struggling with code generation in the REPL. |
| `cost_variance` | Ratio of actual cost to expected median cost for the task's complexity class | Flags outlier runs early. A run at 3x median cost is likely in a redundant verification loop. |

These signals could be implemented as custom signals in `rein.toml` if the RLM framework exposes the necessary telemetry. The `RLMLogger` JSONL output contains sub-call counts and execution traces, which a post-processing script could parse into signal results.

### Integration Path

The most practical integration would be:

1. Rein invokes the agent normally (Claude Code, Codex, etc.)
2. The agent internally uses RLM wrapping for long-context subtasks
3. After completion, rein runs a custom signal that parses the RLM log file
4. The custom signal checks sub-call count, error rate, and cost against configurable thresholds

This keeps RLM awareness outside core rein -- it is a project-specific quality check, not a built-in signal.

---

## 6. SWE-bench Context

### Current Benchmark Landscape

From Rein's benchmark research (07_agent_evaluation_benchmarks.md):

| Benchmark | Top Score | Task Type | Status |
|-----------|-----------|-----------|--------|
| SWE-bench Verified | ~81% | Single bug fixes | Retired (contamination) |
| SWE-bench Pro | ~46% | Multi-file, multi-lang | Active |
| FeatureBench | ~11% | Feature development | Active |
| SWE-EVO | ~21% | Software evolution | Active |

Claude Opus 4.5 leads SWE-bench Pro at 45.9%. No model exceeds 11% on FeatureBench.

### No RLM-Specific Coding Agent Benchmarks

No existing benchmark tests RLM-powered agents on coding tasks. CodeQA is read-only comprehension. SWE-bench tasks typically involve codebases under 100K tokens -- well within context windows, where RLMs show no advantage over base models. FeatureBench and SWE-EVO involve larger scopes but have not been tested with RLM agents.

### What Benchmarks Would Test RLM Agents on Rein-Style Tasks?

A meaningful RLM coding agent benchmark would need:

1. **Large codebases (500K+ tokens)** -- below this threshold, base models with full context perform comparably. The codebase must be large enough that the agent cannot fit it in a single context window.

2. **Multi-file modifications** -- tasks requiring changes across many files test whether the RLM's programmatic navigation translates to correct, coordinated edits. Single-file patches do not exercise the RLM advantage.

3. **Action-oriented evaluation** -- test suites that run against the modified code, not Q&A accuracy. The benchmark must verify that the agent's changes produce working software.

4. **Variance-aware scoring** -- given the 0/6-to-6/6 variance, a meaningful benchmark must run each task multiple times (at minimum 5) and report both median accuracy and variance. A single run per task is unreliable.

5. **Cost tracking** -- report total API cost including all sub-calls, not just the top-level invocation. Without this, cost-effectiveness comparisons are impossible.

No such benchmark exists as of March 2026. The closest candidate would be a modified SWE-bench Pro with larger repository contexts and mandatory multi-run scoring. Building this is a research contribution in itself.

---

## Summary of Gaps

| Gap | Evidence | Impact on Rein |
|-----|----------|-------------------|
| CodeQA is read-only; rein tasks modify code | arXiv:2512.24601 benchmark design | RLM comprehension gains may not transfer to action-oriented tasks |
| Sub-call tokens invisible to budget tracker | Rein monitors top-level agent stream only | Budget EXCEEDED may not trigger on runaway RLM costs |
| 0/6-to-6/6 run variance | arXiv:2603.02615, 30 identical runs | Single-run rein evaluation is unreliable for RLM agents |
| No RLM coding agent benchmark exists | Survey of SWE-bench, FeatureBench, SWE-EVO | Cannot empirically validate RLM value for rein-style tasks |
| Quality gate lacks RLM-specific signals | QUALITY_GATE.md signal inventory | Cannot detect excessive sub-calls, REPL errors, or cost spikes |
| Simple tasks degrade with RLM | arXiv:2603.02615, S-NIAH results | Rein would need a complexity classifier to decide when to engage RLM |

The evidence supports RLMs as a promising approach for tasks involving large codebases and complex reasoning. But the gap between benchmark performance (read-only Q&A on 900K-token repos) and production coding workflows (action-oriented, binary-scored, budget-constrained) remains unbridge by empirical data. Until RLM-specific coding agent benchmarks exist and rein can observe sub-call token usage, integration should be treated as experimental.
