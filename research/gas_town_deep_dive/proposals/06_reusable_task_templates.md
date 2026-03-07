# Proposal: Reusable Task Templates (Formula Pattern)

**Target:** TASKS.md (new section)
**Priority:** Next (when multi-task mode is built)
**Effort:** Low-Medium

---

## Proposed Section: "Task Templates (Planned)"

The following section is proposed for addition to TASKS.md, after the "Workflow Persistence" section (proposal 05).

---

### Task Templates (Planned)

> **Status: Planned.** Requires multi-task sequential execution and workflow persistence.

A task template is a predefined workflow for a common operation (feature development, bug fix, refactoring). Templates reduce the overhead of defining multi-task workflows by providing tested sequences that the operator customizes rather than builds from scratch.

#### Template Format

Templates are YAML files in rein configuration directory:

```yaml
# templates/feature.yaml
template_id: feature
description: "Standard feature development workflow"
parameters:
  - name: feature_name
    required: true
  - name: module
    required: true
  - name: test_command
    default: "pytest tests/ -v"

tasks:
  - task_id: "design-${feature_name}"
    description: |
      Design the ${feature_name} feature for the ${module} module.
      Create a design document at docs/${feature_name}.md with:
      - API surface (function signatures, types)
      - Data model changes
      - Integration points with existing code
    agent: claude-code
    model: opus
    validation_commands:
      - "test -f docs/${feature_name}.md"

  - task_id: "implement-${feature_name}"
    depends_on: ["design-${feature_name}"]
    description: |
      Implement ${feature_name} in the ${module} module following
      the design in docs/${feature_name}.md.
    agent: claude-code
    model: sonnet
    validation_commands:
      - "${test_command}"

  - task_id: "test-${feature_name}"
    depends_on: ["implement-${feature_name}"]
    description: |
      Add comprehensive tests for ${feature_name} in ${module}.
      Cover edge cases, error paths, and integration with existing features.
    agent: claude-code
    model: sonnet
    validation_commands:
      - "${test_command}"
      - "python -m coverage report --fail-under=80"
```

#### Instantiation

```bash
rein workflow --template feature \
  --param feature_name=jwt-auth \
  --param module=src/auth \
  --param test_command="pytest tests/test_auth.py -v"
```

This expands the template into a concrete workflow definition (per proposal 05) with parameter substitution applied.

#### Built-in Templates

Rein ships with templates for common workflows:

| Template | Tasks | Use Case |
|----------|-------|----------|
| `feature` | design → implement → test | New feature development |
| `bugfix` | reproduce → fix → regression-test | Bug fix with regression prevention |
| `refactor` | snapshot-tests → refactor → validate | Refactoring with test-first safety net |
| `migrate` | analyze → migrate → validate → cleanup | Data or API migration |

#### Operator Customization

- Override any task field at instantiation (agent, model, validation commands)
- Add or remove tasks from the template sequence
- Define custom templates in the project's `.rein/templates/` directory

---

## Rationale

Gas Town's Formulas are TOML-based high-level specifications that "cook" into Protomolecules (workflow templates), which instantiate into Molecules (running workflows). This three-layer abstraction is over-engineered, but the core concept — reusable workflow definitions for common operations — is sound.

Gas Town users report that defining tasks from scratch for every feature is tedious and error-prone. Formulas reduce this to "pick a workflow type, fill in parameters." The consistency also improves quality: a tested workflow template produces more reliable results than ad-hoc task definitions.

Rein adaptation collapses Gas Town's three layers (Formula → Protomolecule → Molecule) into one:

| Gas Town | Rein |
|----------|---------|
| Formula (TOML spec) | Template (YAML with parameters) |
| Protomolecule (compiled template) | — (not needed) |
| Molecule (running instance) | Workflow (instantiated from template) |

The template format uses YAML (consistent with rein configuration) with `${parameter}` substitution — simple, readable, and requiring no compilation step.

The built-in templates encode a key Gas Town insight: **multi-agent workflows benefit from role-based model selection.** The `feature` template uses Opus for design (where reasoning quality matters) and Sonnet for implementation (where speed matters). This matches rein's planned per-task `agent`/`model`/`effort` fields.

## Source References

- [Gas Town Task Decomposition](../04_task_decomposition.md) — Section 3 (Formulas, hierarchical task structures), Section 6 (template recommendations)
- [Gas Town Architecture](../01_architecture.md) — Section 4 (MEOW stack, Formula → Protomolecule → Molecule)
- [Gas Town Critical Analysis](../05_critical_analysis.md) — Section 5.2 (Formula templates worth studying)
- TASKS.md — per-task `agent`, `model`, `effort` fields
- ARCHITECTURE.md — planned multi-agent composition (different agents for different roles)
