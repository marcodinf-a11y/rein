# Proposal: Worktree Session Isolation

**Target:** TASKS.md (expand "worktree" subsection under "Workspace Types")
**Priority:** P0 (R1)
**Effort:** Medium

---

## Proposed Section: Enhanced Worktree Specification

The following replaces the existing `### worktree` subsection under "Workspace Types" in TASKS.md.

---

### `worktree`

Creates a git worktree from an existing repository. The agent works on a detached branch — the original working tree is untouched.

- `source` is required and must point to a git repository
- Rein runs `git worktree add <sandbox_path> -b rein/<task_id> HEAD` from the source repo
- Seed files from `files` are written into the worktree (overwriting existing files or adding new ones)
- `setup_commands` run after seeding
- Diff baseline: the commit at worktree creation time (`HEAD` of source at invocation)
- Cleanup: `git worktree remove <sandbox_path>` and delete branch (see Branch Lifecycle below)

Best for: brownfield tasks against git repositories — refactoring, bug fixes, feature additions. The agent has access to the full project structure, existing tests, and dependencies.

#### Branch Naming

The default branch name is `rein/<task_id>`. The prefix is configurable:

```yaml
# config.yaml (global or project)
workspace:
  worktree:
    branch_prefix: "rein/"  # default, configurable
```

This avoids the hardcoded prefix problem documented in T3Code (issue #272). Teams can use `ai/`, `agent/`, or any prefix that fits their branch naming convention.

#### Worktree Lifecycle

```
1. Pre-flight    Verify source is a git repo
                 Verify branch name doesn't already exist
                 Verify no dirty state that would prevent worktree creation
                 │
2. Create        git worktree add -b rein/<task_id> <sandbox_path> HEAD
                 │
3. Seed          Write files from task definition
                 Run setup_commands
                 │
4. Execute       Start agent subprocess with cwd = sandbox_path
                 Monitor context pressure
                 │
5. Capture       Collect diff, artifacts, report
                 Auto-commit if terminated by rein (wrap-up protocol)
                 │
6. Cleanup       git worktree remove <sandbox_path>
                 Branch handling per cleanup_branch setting
```

#### Branch Cleanup

After task completion, the worktree is always removed. Branch handling depends on the outcome:

| Outcome | Branch Action | Rationale |
|---------|--------------|-----------|
| Validation passed | Keep branch | User may want to review and merge |
| Validation failed | Keep branch | Preserves partial work for debugging |
| Killed (context pressure) | Keep branch | Preserves partial work from wrap-up |
| Killed (timeout) | Keep branch | Same reasoning |
| Error (adapter failure) | Delete branch | No useful work to preserve |

Stale branches accumulate over time. `rein clean` removes worktree branches that are fully merged or older than a configurable threshold:

```bash
rein clean --branches              # remove merged rein/* branches
rein clean --branches --age 7d     # also remove unmerged branches older than 7 days
```

#### Worktree Limitations

- **Shared `.git` directory**: All worktrees share the same `.git` — concurrent worktrees on the same repo can conflict on index operations. Rein's current sequential execution avoids this; parallel dispatch (future) will need worktree locking.
- **Submodules**: `git worktree add` does not initialize submodules. If the source repo uses submodules, `setup_commands` should include `git submodule update --init`.
- **Ports and global state**: Worktree isolation is filesystem-level only. If setup commands start servers on fixed ports, concurrent worktrees will conflict.

---

## Rationale

Rein's TASKS.md documents worktree as a workspace type but doesn't specify branch naming, lifecycle, cleanup, or known limitations. T3Code's production worktree implementation (889 commits, actively used) provides concrete patterns:

1. **Branch naming convention** — T3Code uses `t3code/{8-hex}` but hardcoded the prefix (issue #272), making it unusable for teams with branch naming policies. Rein should make the prefix configurable from day one.

2. **Lifecycle stages** — T3Code's flow (validate → create → setup → dispatch → cleanup) maps directly to Rein's session lifecycle but adds pre-flight validation (branch exists? dirty state?) that the current spec omits.

3. **Branch cleanup strategy** — T3Code leaves branches until the user explicitly closes a thread. Rein should be more aggressive: always remove the worktree, keep branches by default (user merges manually), provide `rein clean` for housekeeping.

4. **Known limitations** — T3Code's documentation doesn't warn about shared `.git` conflicts or submodule behavior. Documenting these upfront prevents debugging surprises.

## Source References

- [T3Code Deep Dive: Workspace Management](../05_workspace_management.md) — worktree strategy, `GitCore.ts`, branch naming, issue #272
- [T3Code Deep Dive: Synthesis](../00_synthesis.md) — §5 (worktree isolation production-validated)
- [T3Code Deep Dive: Critical Analysis](../07_critical_analysis.md) — §4 (adopt worktree patterns), §2 (hardcoded prefix weakness)
- TASKS.md — current worktree specification (3 lines, no lifecycle/cleanup/limitations)
