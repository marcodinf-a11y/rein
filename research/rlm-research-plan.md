# RLMs (Recursive Language Models) - Research Plan

Paper: https://arxiv.org/abs/2512.24601

## Phase 1: Core Paper Understanding
- [x] Read the full RLMs paper end-to-end
- [x] Extract the core thesis: what problem do RLMs solve and why existing approaches fall short?
- [x] Map the architecture: how does the recursive decomposition work?
- [x] Understand the REPL integration: how is Python REPL used as the execution environment?
- [x] Document the "offloading context into variables" mechanism — how does this differ from standard context window usage?
- [x] Identify the key claims and reported results/benchmarks

## Phase 2: Technical Deep Dive
- [x] Analyze the recursion strategy: what determines when/how to decompose vs. solve directly?
- [x] Study the memory model: how are intermediate results stored and referenced across recursive calls?
- [x] Examine the code generation aspect: what kind of code does the LM produce in the REPL?
- [x] Understand error handling: what happens when generated code fails?
- [x] Identify the base cases / termination conditions for recursion
- [x] Review the prompt templates and system prompts used

## Phase 3: Comparison with Related Work
- [x] Compare with ReAct (reasoning + acting loop) — what does RLM add?
- [x] Compare with REPL-Plan (code-augmented planning) — overlaps and differences
- [x] Compare with standard tool-use / function-calling agents
- [x] Compare with chain-of-thought and tree-of-thought approaches
- [x] Compare with CodeAct and similar code-as-action frameworks
- [x] Position RLMs in the broader landscape of agentic code execution

## Phase 4: Practical Implications
- [x] Assess applicability to rein
- [x] Identify which tasks/benchmarks RLMs excel at vs. struggle with
- [x] Evaluate computational overhead of recursive calls
- [x] Consider token efficiency: does REPL offloading actually save tokens?
- [x] Identify limitations and failure modes reported in the paper
- [x] Explore potential extensions or modifications

## Phase 5: Implementation Considerations
- [x] Review any reference implementation or open-source code
- [x] Identify key abstractions needed: recursive call manager, REPL sandbox, variable store
- [x] Consider safety implications of arbitrary code execution in a loop
- [x] Evaluate how RLM patterns could integrate with existing agent frameworks
- [x] Note any model-specific requirements (context length, code generation quality)

## Key Questions to Answer
1. How does RLM handle unbounded context differently from chunking or summarization?
2. What is the recursion depth in practice — does it stay shallow or go deep?
3. Is the approach model-agnostic or does it depend on specific LM capabilities?
4. How does performance scale with task complexity?
5. What are the failure modes and how gracefully does it degrade?
