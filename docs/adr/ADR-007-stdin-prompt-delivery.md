# ADR-007: Stdin-Based Prompt Delivery

## Status

Accepted

## Context

Rein assembles a multi-section prompt (context preamble, task body, deviation rules, completion signal) and delivers it to the agent CLI. Two delivery mechanisms:

- **CLI argument:** Pass the prompt as a positional argument or via a flag (e.g., `claude -p "prompt"`). Simple, but subject to `ARG_MAX` limits on Linux (~2MB). Assembled prompts with large file seeds or detailed instructions can exceed this. Also visible in `ps` output.
- **Stdin pipe:** Pipe the prompt to the agent's stdin (`echo "prompt" | claude -p`). No size limit. Not visible in process listings. All three agent CLIs accept stdin input.

Claude also supports `--append-system-prompt` for injecting instructions into the system prompt. This splits delivery into two channels (system prompt + user message), complicating debugging — the operator cannot see the full assembled prompt in one place.

## Decision

All prompts are delivered via stdin pipe for all three agents. The prompt is written to the subprocess's stdin via `asyncio.create_subprocess_exec` with `stdin=PIPE`. No CLI arguments carry prompt content.

Single delivery channel: the entire assembled prompt (all sections) goes as one stdin message. No split between system and user prompts.

## Consequences

**Positive:**

- No `ARG_MAX` limits. Prompts of any size work.
- Consistent across all three agents — one delivery mechanism, one code path.
- The full assembled prompt is a single string, easy to log and debug.
- Not visible in `ps` output (minor security benefit for prompts containing repo context).

**Negative:**

- Stdin delivery means the prompt is a user message, not a system message. The agent may weight it differently than system-level instructions. In practice, all three CLIs in headless mode treat stdin as the primary instruction.
- Cannot use `--append-system-prompt` (Claude) for higher-priority instruction injection. If prompt compliance becomes an issue, this tradeoff may be revisited.
