# ADR-008: JSON Task Format for MVP

## Status

Accepted

## Context

Task definitions need a file format. The two candidates:

- **JSON:** Strict syntax, universal tooling, highest structured output reliability across LLMs (99.36% from StructEval benchmarks). No comments, no multiline block scalars, verbose for long prompts.
- **YAML:** Comments, multiline block scalars (`|`), ~24% fewer tokens for equivalent content. But indentation-sensitive, implicit type coercion (`yes` → `true`, `3.0` → float), and lower LLM generation reliability.

Both formats can express the same task schema. The question is which to ship first and whether to support both simultaneously.

## Decision

MVP ships with JSON task format only. YAML support is deferred to Release 2.

JSON is the canonical format: the task schema, all examples, and the validation logic target JSON. When YAML lands, it will be parsed into the same internal `TaskDefinition` dataclass — JSON remains the reference.

## Consequences

**Positive:**

- Strict syntax catches errors at parse time (missing commas, trailing commas, unquoted strings). No silent type coercion.
- LLMs generating task definitions produce valid JSON more reliably than valid YAML.
- One parser, one schema, one set of examples to maintain for MVP.
- Tooling (jq, IDE validation, JSON Schema) is mature and universal.

**Negative:**

- Long prompts in JSON require escaped newlines (`\n`) or concatenated strings. This is less readable than YAML's block scalars.
- No comments in JSON. Operators cannot annotate task definitions inline (must use a `metadata` or `notes` field).
- YAML's ~24% token savings on prompts are deferred. Not relevant at MVP scale but matters when task libraries grow.
