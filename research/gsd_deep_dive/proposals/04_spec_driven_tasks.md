# Proposal: Spec-Driven Task Definitions

**Priority:** Next (when auto-generation exists) | **Effort:** Medium | **Source:** GSD Deep Dive

---

## Summary

If rein ever supports auto-generated task definitions (e.g., an LLM decomposes a project into tasks), implement a validation pass before dispatch — similar to GSD's planner-checker verification loop.

## Rationale

GSD's planner creates plans, then a separate checker agent verifies them (up to 3 revision iterations). This catches:
- Missing requirement coverage
- Vague or unactionable task descriptions
- Incorrect dependency declarations
- Plans too large for fresh context windows

The rein currently uses human-written task definitions, which are inherently validated by the operator's understanding. If rein adds auto-generation, the same validation is needed.

## Proposed Design

### Pre-Dispatch Validation (for auto-generated tasks)

Before dispatching auto-generated tasks, validate:

1. **Completeness:** Every stated goal has at least one task
2. **Actionability:** Each task prompt is specific enough to execute (not "implement the feature")
3. **Scope sizing:** Each task's estimated scope fits within the token budget
4. **Dependency correctness:** No circular dependencies, all referenced task IDs exist
5. **Validation coverage:** Each task has at least one validation command

### Implementation Options

**Option A: Programmatic validation (recommended)**
- Schema validation of task structure
- Dependency graph cycle detection
- Prompt length heuristic for scope sizing
- No LLM cost

**Option B: LLM-based checker (GSD's approach)**
- Separate agent reviews generated tasks
- Higher quality checks (semantic actionability)
- Additional LLM cost per validation pass

### Requirement Traceability (Optional)

GSD's `requirements` field in plan frontmatter enables tracing which plans address which requirements. The rein's `tags` field already serves this purpose at a coarser granularity. If more formal traceability is needed, add a `requirements` field to TaskDefinition:

```python
requirements: list[str] = field(default_factory=list)  # Requirement IDs this task addresses
```

## Scope

This proposal is contingent on rein supporting auto-generated task definitions. If tasks remain human-written, this validation is unnecessary — the operator is the validator.

## Target Documents

- TASKS.md — add validation section for auto-generated tasks
- ARCHITECTURE.md — add pre-dispatch validation to execution flow

## Verification

- [ ] Validation catches: missing validation commands, circular deps, empty prompts
- [ ] Validation runs before dispatch, not after
- [ ] Human-written tasks bypass validation (operator is the validator)
