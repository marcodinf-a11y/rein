# T3Code Deep Dive: Workspace Management

**March 2026**

This document analyzes T3Code's workspace isolation strategy, focusing on its git worktree management, branch naming conventions, session isolation model, and frontend orchestration flow. Comparison with Rein's multi-mode workspace approach is provided throughout.

---

## 1. Architecture Overview

T3Code uses git worktrees as its sole workspace isolation mechanism. Every coding session (called a "thread" in T3Code's UI) gets its own worktree, ensuring that concurrent work does not produce file-level conflicts in the main working tree.

The implementation spans two layers:

| Layer | File | Responsibility |
|-------|------|---------------|
| **Git operations** | `GitCore.ts` | Low-level worktree CRUD, branch management |
| **Branch utilities** | `packages/shared/src/git.ts` | Name sanitization, auto-naming |
| **Frontend orchestration** | Thread creation flow | Validation, setup scripts, UI locking |

This is a simpler architecture than Gas Town's Polecat/Refinery pattern or Ralph Orchestrator's worktree-per-loop model. T3Code treats worktrees as disposable sandboxes, not as long-lived coordination primitives.

---

## 2. GitCore.ts — The Worktree Engine

`GitCore.ts` is the central module for all git operations. The workspace-relevant methods are:

### 2.1 createWorktree

Creates a new worktree with an associated branch:

```
git worktree add -b [branch] [path]
```

The `-b` flag creates a new branch at the same time as the worktree. This is a single atomic operation — the branch and worktree are born together. If branch creation fails (name collision, invalid characters), the worktree is never created.

The worktree path is derived from a session identifier, typically placed in a temporary or project-local directory. Each worktree is a full checkout of the repository at the branch point, with its own index, HEAD, and working tree.

### 2.2 removeWorktree

Tears down a worktree after work completes or the session is abandoned. This calls `git worktree remove` with force semantics when necessary. The associated branch is not automatically deleted — it persists in the repository until explicitly cleaned up or renamed.

### 2.3 listBranches

Enumerates all branches, used to detect naming collisions before worktree creation and to present the user with existing work branches.

### 2.4 checkoutBranch / renameBranch

`checkoutBranch` switches the worktree's HEAD to a different branch. `renameBranch` handles the transition from temporary branch names to semantic names after work completes (see Section 3).

---

## 3. Branch Naming Strategy

T3Code uses a two-phase branch naming scheme:

### Phase 1: Temporary Name

When a thread starts, the branch is created with the pattern:

```
t3code/{8-hex-chars}
```

The 8-character hex suffix is generated randomly (likely `crypto.randomBytes(4).toString('hex')` or equivalent). Examples: `t3code/a1b2c3d4`, `t3code/f0e9d8c7`.

This temporary name serves as a session identifier. It is:
- **Collision-resistant**: 4 billion possible values make conflicts negligible in practice
- **Non-descriptive**: Carries no semantic information about the work being performed
- **Identifiable**: The `t3code/` prefix marks it as machine-generated

### Phase 2: Semantic Rename

After work completes, T3Code renames the branch to something meaningful — either derived from the task description or chosen by the user. The `renameBranch` method in `GitCore.ts` handles this transition.

The rename-after-work pattern avoids the problem of naming a branch before knowing what the work actually produces. Many developers have experienced the mismatch between `fix/auth-bug` and a branch that ended up restructuring the entire auth module.

### Branch Sanitization

`packages/shared/src/git.ts` provides two utilities:

- **sanitizeBranchFragment**: Normalizes arbitrary strings into valid git branch name components. Strips characters illegal in git refs (`~`, `^`, `:`, `?`, `*`, `[`, `\`, spaces, leading dots, trailing `.lock`), collapses consecutive slashes and dots, and trims to a reasonable length.

- **resolveAutoFeatureBranchName**: Generates a complete branch name from a task description or prompt. Likely combines a prefix with a sanitized version of the description. This is used when the user does not manually name the branch.

### Issue #272: Hardcoded Prefix Problem

The `t3code/` prefix is hardcoded. Issue #272 documents that this is not configurable, which creates problems for:

- **Teams with branch naming conventions**: Organizations enforcing patterns like `feature/`, `bugfix/`, or `username/` cannot integrate T3Code branches into their workflow without manual renaming
- **CI/CD pipelines**: Branch-based triggers that filter on prefix patterns will not recognize T3Code branches unless explicitly configured
- **Multi-tool environments**: If other tools also use hardcoded prefixes, the branch namespace becomes cluttered with tool-specific conventions

This is a design oversight, not a fundamental limitation. The fix is straightforward — expose the prefix as a configuration option — but its presence in the issue tracker suggests it has caused real friction for users.

---

## 4. Session Isolation Model

### One Thread, One Worktree

T3Code's isolation guarantee is simple: each thread (a unit of work in the UI) gets its own worktree. Two threads never share a worktree. This means:

- **No file conflicts**: Thread A writing to `src/auth.ts` does not affect Thread B's copy of `src/auth.ts`
- **Independent git state**: Each worktree has its own HEAD, index, and staged changes
- **Clean rollback**: Abandoning a thread means removing the worktree and optionally deleting the branch — no partial changes leak into the main tree

### What Isolation Does Not Cover

Worktree isolation is filesystem-level only. It does not protect against:

- **Shared resources**: Database connections, network ports, external services. If Thread A starts a dev server on port 3000, Thread B cannot use the same port.
- **Global state**: Package managers with global caches, environment variables, system-level configuration files outside the repository
- **Build artifacts**: If the build system writes to locations outside the worktree (e.g., a shared `node_modules` hoisted to a monorepo root), conflicts can occur
- **Git operations on shared refs**: Both worktrees share the same `.git` directory. Operations like `git gc`, ref packing, or concurrent pushes to the same remote branch can interfere.

These limitations are inherent to git worktrees, not specific to T3Code. Any worktree-based isolation system (including Rein's worktree mode) faces the same constraints.

---

## 5. Frontend Worktree Flow

The frontend orchestration follows a strict sequence when creating a new coding session:

### Step 1: Validate

Before creating a worktree, T3Code validates:
- The repository is a valid git repo (not bare, not in a detached state that prevents branching)
- The target branch name does not already exist (checking via `listBranches`)
- Sufficient disk space exists for a full checkout (worktrees duplicate the working tree, though they share the object store)

### Step 2: Create

Calls `createWorktree` to atomically create the branch and worktree. If this fails, the UI reports the error and does not proceed. No partial state is left behind.

### Step 3: Run Setup Scripts

After the worktree is created, T3Code runs project-specific setup scripts. These handle:
- Installing dependencies (`npm install`, `pip install -r requirements.txt`, etc.)
- Building prerequisite artifacts
- Configuring environment files
- Any project-specific initialization

Setup scripts run in the worktree's directory, ensuring they operate on the isolated copy. This step is critical — a bare checkout without dependencies is not a functional development environment.

### Step 4: Dispatch Turn

With the worktree ready and dependencies installed, T3Code dispatches the first "turn" (a unit of agent work) to the coding agent. The agent operates exclusively within the worktree's filesystem.

### Step 5: Lock UI Until Complete

The frontend locks the thread's UI while a turn is in progress. The user cannot modify the prompt, send new instructions, or interact with the worktree until the agent completes or is cancelled. This prevents race conditions between human edits and agent edits.

The lock is per-thread, not global. Other threads (each with their own worktree) remain interactive.

### Flow Diagram

```
User clicks "New Thread"
        |
        v
  [Validate repo + branch name]
        |
        v
  [git worktree add -b t3code/a1b2c3d4 /path/to/worktree]
        |
        v
  [Run setup scripts in worktree]
        |
        v
  [Dispatch first turn to agent]
        |
        v
  [Lock thread UI] -----> [Agent works in worktree] -----> [Unlock UI]
        |
        v
  [User reviews, iterates, or merges]
        |
        v
  [Rename branch to semantic name]
        |
        v
  [Remove worktree when done]
```

---

## 6. Comparison with Rein's Workspace Model

Rein supports three workspace types. T3Code supports one. The differences are significant:

### 6.1 Workspace Type Comparison

| Property | T3Code (worktree) | Rein: tempdir | Rein: worktree | Rein: copy |
|----------|-------------------|---------------|----------------|------------|
| **Git integration** | Full (branch per session) | None | Full (branch per task) | Full (independent clone) |
| **Disk cost** | Low (shared objects) | Minimal | Low (shared objects) | High (full copy) |
| **Setup speed** | Fast (checkout only) | Instant | Fast (checkout only) | Slow (full copy + install) |
| **Isolation level** | Filesystem | Filesystem + no git | Filesystem | Filesystem + git |
| **Use case** | Feature work | Scratch/evaluation | Feature work | Untrusted or destructive work |
| **Merge path** | Branch merge | N/A (disposable) | Branch merge | Cherry-pick or patch |
| **Shared .git risks** | Yes | No | Yes | No |

### 6.2 Why Multi-Mode Matters

T3Code's worktree-only approach works for its primary use case: feature development on existing repositories. But it fails for:

- **Evaluation tasks**: Running benchmarks, testing prompts, or exploring approaches does not need (and should not create) git branches. Rein's tempdir mode handles this cleanly.
- **Destructive experimentation**: If the agent might corrupt the repository state (e.g., force-push, rebase experiments, `.git` directory manipulation), worktree isolation is insufficient because worktrees share the `.git` directory. Rein's copy mode provides full isolation.
- **Non-git projects**: Not every codebase uses git. T3Code cannot operate on projects without a git repository. Rein's tempdir and copy modes work regardless of version control.
- **CI environments**: Many CI pipelines check out code in detached HEAD state or shallow clones where `git worktree add` may fail or behave unexpectedly. Rein's tempdir and copy modes are more robust in these environments.

### 6.3 What T3Code Does Better

T3Code's single-mode approach has advantages:

- **Simplicity**: One code path means fewer bugs, fewer configuration options, fewer edge cases
- **Consistent merge story**: Every session produces a branch that can be merged via standard git workflow (PR, merge, rebase)
- **Branch-as-history**: The temporary-to-semantic rename pattern creates a clean audit trail of what the agent did and when
- **No mode selection burden**: The user never has to choose a workspace type — there is only one

Rein's three-mode approach trades simplicity for flexibility. The cost is a configuration decision the user must make (or a default must be chosen). The benefit is correct behavior in a wider range of scenarios.

---

## 7. Worktree Lifecycle Gaps

### 7.1 Cleanup

T3Code's `removeWorktree` handles the happy path. The failure modes are:

- **Orphaned worktrees**: If T3Code crashes between creating a worktree and recording its existence, the worktree persists on disk with no reference in T3Code's state. `git worktree prune` can clean these up, but T3Code does not appear to run this automatically.
- **Locked worktrees**: If a process holds a file handle in the worktree (a running dev server, a file watcher, an editor), `git worktree remove` fails. The `--force` flag helps but does not handle all cases.
- **Branch accumulation**: Temporary `t3code/*` branches accumulate if sessions are abandoned without cleanup. Over time, `git branch` output becomes cluttered.

### 7.2 Concurrent Worktree Limits

Git does not impose a hard limit on worktree count, but practical limits exist:

- **Disk space**: Each worktree duplicates the working tree (not the object store). For a 1 GB working tree, 10 worktrees consume ~10 GB.
- **Inode pressure**: Many small files (e.g., `node_modules`) multiply across worktrees
- **Setup time**: Each worktree needs `npm install` or equivalent, which may take minutes

T3Code does not appear to impose a configurable limit on concurrent worktrees. Rein should consider this in its worktree mode — a sane default (e.g., max 5 concurrent worktrees) prevents resource exhaustion.

---

## 8. Lessons for Rein

### Adopt

1. **Temporary-to-semantic branch rename**: Create branches with generated names, rename after the agent's work reveals the actual scope. This avoids premature naming and produces cleaner git history.

2. **Atomic create (branch + worktree)**: Use `git worktree add -b` to create both in a single operation. This prevents orphaned branches without worktrees or worktrees without branches.

3. **Setup script execution in worktree**: After creating a worktree, run project-specific setup before dispatching work. A bare checkout is not a functional environment.

4. **UI lock during agent execution**: Prevent user modifications to the workspace while the agent is active. In Rein's CLI context, this means clear signaling that the workspace is under agent control.

### Avoid

1. **Hardcoded branch prefix**: Make the branch prefix configurable from the start. Default to `rein/` but allow override via `rein.toml`.

2. **Single workspace mode**: T3Code's worktree-only approach is a limitation, not a feature. Rein's three-mode design is correct.

3. **No worktree cleanup automation**: Implement periodic `git worktree prune` and stale branch detection. Do not rely on the user to clean up.

4. **No concurrent worktree limit**: Set a configurable maximum to prevent disk exhaustion.

### Study Further

1. **Setup script discovery**: How does T3Code determine which setup scripts to run? Is it convention-based (look for `package.json`, `Makefile`, `setup.py`), configuration-based (user specifies), or agent-determined? Rein needs a strategy for this.

2. **Worktree path selection**: Where does T3Code place worktrees on disk? Inside the repo's parent directory? In a temp directory? The choice affects discoverability, cleanup, and disk layout.

---

## 9. Summary

T3Code's workspace management is a clean, single-purpose implementation of git worktree isolation. The architecture is straightforward: `GitCore.ts` wraps git commands, `packages/shared/src/git.ts` handles naming, and the frontend orchestrates the lifecycle. The two-phase branch naming (temporary hex to semantic name) is a pattern worth adopting.

The limitation is scope. Worktree-only isolation fails for non-git projects, destructive experimentation, and evaluation tasks. Rein's three-mode approach (tempdir, worktree, copy) covers these cases at the cost of additional complexity. The tradeoff is worth it — workspace isolation must match the task's requirements, not the tool's implementation constraints.

The hardcoded `t3code/` prefix (Issue #272) is a small but telling example of how single-user design decisions create friction at team scale. Rein should make every convention configurable from the start, with sensible defaults that work without configuration.

---

## Sources

- T3Code source: `GitCore.ts`, `packages/shared/src/git.ts`
- T3Code Issue #272: hardcoded branch prefix
- Git documentation: `git-worktree(1)`
- Rein docs: ARCHITECTURE.md, SESSIONS.md
- Prior deep dives: gas_town_deep_dive/05_critical_analysis.md, ralph_orchestrator_deep_dive/00_synthesis.md
