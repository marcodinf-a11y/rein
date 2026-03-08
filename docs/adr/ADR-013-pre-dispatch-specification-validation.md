# ADR-013: Pre-Dispatch Specification Validation

## Status

Accepted

## Context

The single most documented failure mode across Gas Town, Ralph Loop, and GSD deployments is **vague specifications that waste compute**. Gas Town amplifies this through LLM-driven decomposition (the Mayor), but even operator-defined task sequences (Rein's model) suffer when operators write underspecified tasks.

Gas Town's own users consistently report that specification quality is the bottleneck:

- "The limiting factor is how fast you can think, give direction, and validate" — [Embracing Enigmas](https://embracingenigmas.substack.com/p/exploring-gas-town)
- "Planning and design become the limiting factor. Humans must deeply consider what to build before prompting agents" — [Maggie Appleton](https://maggieappleton.com/gastown)

A review of the Gas Town codebase (github.com/steveyegge/gastown) confirms that **Gas Town has zero pre-dispatch specification quality checks**. Their validation is purely structural and operational — valid target paths, bead status, rig state, file locks. The `AcceptanceCriteria` field exists on beads but is entirely optional and never gated on. This gap costs Gas Town users $2,000-5,000/month in wasted work from vague specs producing merge conflicts and redundant effort.

Rein can address this gap with a lightweight deterministic gate that catches obvious omissions before any tokens are spent.

### Options considered

**Option A — Boolean checks only.** Two deterministic checks: prompt length and validation commands presence. Simple, fast, honest about what it measures. Doesn't pretend to assess spec quality — just catches degenerate cases.

**Option B — Scored heuristics.** A weighted score (0-100) combining prompt length, validation commands, file path references, structural markers, and action verbs. More sophisticated but implies a quality measurement we can't actually deliver. Operators would game the score, and gaming it doesn't guarantee better specs.

**Option C — LLM-based analysis.** Use an LLM to assess specification quality. High cost, introduces its own failure modes, and spending tokens to validate whether you should spend tokens is circular.

## Decision

**Option A — Boolean checks only.**

Two checks, both deterministic and sub-millisecond:

| Check | Rule | Failure prevented |
|-------|------|-------------------|
| **Prompt length** | `len(prompt) >= 50` | One-liner prompts ("fix the auth") that produce ambiguous agent behavior |
| **Validation commands** | `len(validation_commands) > 0` | Tasks without validation commands make the quality gate meaningless |

### Modes

Controlled by `--spec-check` CLI flag or `[spec_validation] mode` in `rein.toml`:

| Mode | On failure | Non-interactive (no TTY or `--yes`) |
|------|-----------|--------------------------------------|
| `warn` (default) | Print warning, prompt `[y/N]` | Print warning, dispatch anyway |
| `strict` | Print error, exit non-zero | Print error, exit non-zero |
| `skip` | No checks run | No checks run |

### CLI flags

- `--spec-check {warn,strict,skip}` — override validation mode (default: `warn`)
- `--yes` / `-y` — auto-confirm all interactive prompts, including spec validation warnings

### Configuration

```toml
[spec_validation]
mode = "warn"                      # "warn" | "strict" | "skip"
min_prompt_length = 50             # minimum prompt character count
require_validation_commands = true # require non-empty validation_commands
```

### Resolution order

CLI `--spec-check` flag > `rein.toml` `[spec_validation] mode` > default (`warn`).

The `--yes` flag affects only the interactive prompt in `warn` mode — it does not change the validation mode itself.

### Warning output

```
Warning: Task "refactor-auth" has specification issues:
  - No validation commands defined (quality gate will be meaningless)
  - Prompt is 23 characters (minimum: 50 for reliable execution)

Dispatch anyway? [y/N]
```

In strict mode, the same output is printed as an error and dispatch is blocked:

```
Error: Task "refactor-auth" failed specification validation:
  - No validation commands defined (quality gate will be meaningless)
  - Prompt is 23 characters (minimum: 50 for reliable execution)
```

### Pipeline position

```
Load JSON → Parse TaskDefinition → Spec Validation → Resolve agent/model → Check agent available → Create sandbox → ...
```

Spec validation runs immediately after parsing, before any I/O, subprocess, or sandbox work. This is the cheapest possible position in the pipeline.

### Batch behavior

When `rein run -t tasks/` dispatches multiple tasks, each task is validated independently. In `warn` mode with a TTY, each failing task prompts separately. In `warn` mode without a TTY (or with `--yes`), all warnings are printed and all tasks dispatch. In `strict` mode, the first failing task aborts the entire run.

## What is explicitly not included

- **Acceptance criteria heuristics.** Regex-matching for keywords like "should"/"must" is brittle theater. Gas Town has an `AcceptanceCriteria` field and a `HasUncheckedCriteria()` function — neither prevents vague specs.
- **LLM-based spec analysis.** Costs tokens, introduces failure modes, circular logic.
- **Automatic spec improvement.** The operator writes specs. Rein catches omissions, not quality.
- **Semantic validation.** No check that the prompt "makes sense." Only structural checks.

## Consequences

- Operators get early feedback on underspecified tasks before wasting tokens
- The quality gate remains meaningful — tasks without validation commands are flagged
- Zero runtime cost (deterministic string checks, no subprocess or LLM invocation)
- `min_prompt_length` is a configurable proxy, not a quality guarantee — operators can override via config
- Projects can enforce strict validation in CI by setting `mode = "strict"` in `rein.toml`

## Source References

- [Gas Town Deep Dive — Task Decomposition](../../research/gas_town_deep_dive/04_task_decomposition.md) — specification quality as bottleneck
- [Gas Town Deep Dive — Proposal 03](../../research/gas_town_deep_dive/proposals/03_specification_validation.md) — original proposal
- [Gas Town GitHub](https://github.com/steveyegge/gastown) — confirmed zero pre-dispatch spec quality checks
- [GSD Deep Dive](../../research/gsd_deep_dive/00_synthesis.md) — spec validation for auto-generated tasks
- [MVP Value Synthesis](../../research/MVP_VALUE_SYNTHESIS.md) — Tier 1, Item 5
