# ADR-001: Agent Git Identity

## Status

Accepted

## Context

When rein runs an agent in a sandbox, any git commits the agent makes use the operator's global git config. This means:

- Git history doesn't distinguish agent work from human work. In worktree sandboxes where changes are merged back, agent commits appear as if the human wrote them.
- Post-mortem analysis requires cross-referencing timestamps between `git log` and JSON reports to determine which agent produced a commit.
- Multi-agent workflows (planned for Release 3) will need attribution to understand which agent contributed what.

Principal Skinner's agent identity analysis argues "you can only debug what you can identify." OWASP ASI03 (Identity & Privilege Abuse) recommends isolated identities per agent with task-scoped permissions. The full identity infrastructure (per-agent SSH keys, service accounts, credential scoping) is enterprise-scale and irrelevant for local development, but git author attribution is near-zero cost.

## Decision

Every agent subprocess gets git author and committer environment variables set before launch. Both `GIT_AUTHOR_*` and `GIT_COMMITTER_*` are set to the same values so no operator identity leaks into agent commits.

```
GIT_AUTHOR_NAME="{agent}/{model_short}/{effort}"
GIT_AUTHOR_EMAIL="agent@rein.local"
GIT_COMMITTER_NAME="{agent}/{model_short}/{effort}"
GIT_COMMITTER_EMAIL="agent@rein.local"
```

Specific decisions:

- **Environment variables, not `git config --local`.** Env vars work for all three workspace types without requiring a `.git` directory at setup time. They take effect only when git commands actually run.
- **Short model aliases** (`opus`, `sonnet`, `o3`, `flash`, `pro`) for readability in `git log`. Not full model IDs.
- **Effort segment omitted when not specified.** Format is `{agent}/{model}` when effort is default/unset, `{agent}/{model}/{effort}` when explicit. In code, filter out `"default"` — do not emit `claude-code/opus/default`.
- **Fixed email** `agent@rein.local` for all agents. Not a real address. Satisfies git's email requirement and provides a single grep target: `git log --author="agent@rein.local"`.
- **Both AUTHOR and COMMITTER set.** Prevents the operator's global committer identity from appearing on agent commits.
- **Rein wrap-up commits reuse the agent's identity.** The wrap-up captures the agent's uncommitted work, so the same identity applies. The commit message prefix (`rein:`) distinguishes wrap-up commits from agent-authored ones.

## Consequences

**Positive:**

- `git log --author="agent@rein.local"` finds all agent commits across any repo that used rein.
- `git log --author="claude-code/opus"` filters by specific agent/model.
- `git blame` shows which agent wrote each line.
- Consistent with the report's `agent_name`, `model`, and `effort` fields — identity is derived from the same source.
- Multi-agent workflows get attribution for free when they land.

**Negative:**

- Agents that amend or rebase commits will carry the rein-set identity, which is correct but may surprise operators who expect their own identity on rebased work.
- The short model alias requires a mapping table in the adapter. If a model has no short alias, the full ID is used as fallback.

**Neutral:**

- The env vars are scoped to the subprocess — the operator's global git config is never modified.
- For tempdir workspaces where git is never used, the env vars are set but have no effect.
