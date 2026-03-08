# PRD Review — 2026-03-08

Review of `PRD_SKETCH.md` prior to implementation.

---

## Strengths

1. **Traceability** — every FR links back to source doc and target module.
2. **Error handling matrix (S4)** covers the right failure modes with clear behaviors.
3. **Acceptance criteria (S5)** are concrete and testable — three task types covering greenfield/brownfield.
4. **Resolved decisions (S8)** prevents re-litigation of settled questions.
5. **Defense-in-depth framing (S3.2)** with the probabilistic vs. deterministic control distinction is sharp.

---

## Gaps — Missing FRs

### G-1: Retry orchestration — RESOLVED

FR-093 defines the retry cap (default 4) but not the retry loop itself. Resolved through discussion — proposed FRs below.

#### Proposed FRs

| ID | Requirement | Module |
|----|-------------|--------|
| FR-093a | Retry triggers: retry on any non-`completed` termination (`error`, `context_pressure`, `timed_out`) and on `completed` with verdict "fail". The bounded retry cap (FR-093) is the sole guard against waste. | `runner.py` |
| FR-093b | Sandbox state policy: if validation ran and failed (verdict "fail", validation commands returned non-zero), carry forward the sandbox into the next round. On all other outcomes (validation couldn't run, agent crash, context pressure kill, timeout), create a fresh sandbox. | `runner.py`, `sandbox.py` |
| FR-093c | Cross-round knowledge injection: before each retry, inject into the prompt: (1) updated LEARNINGS.md extracted from the failed round (per FR-091a), (2) condensed failure narrative — progress classification, plain-language termination reason, validation output truncated to 30-line excerpt, `git diff --stat` summary of files touched. | `runner.py`, `prompts.py` |
| FR-093d | Operator vs. agent reporting split: the full escalation report (FR-092) is written to the JSON report for the operator. The agent receives only the condensed failure narrative from FR-093c. | `escalation.py`, `prompts.py` |

### G-2: No FR for PROGRESS.md / DEFERRED.md creation

FR-094 mentions these files in prompt injection. FR-062 says "update PROGRESS.md". But no FR covers *creating* them at sandbox setup time — parallel to FR-091's LEARNINGS.md injection.

### G-3: No FR for config file parsing

FR-085/086 describe the two-layer config and resolution order, but no FR covers actually parsing the config file. The format is also inconsistent (see I-1).

### G-4: No FR for workspace source path

FR-011/012 reference "source repo" and "source tree" but no FR or CLI flag specifies how the user provides the source path. Needs a `--source` flag or task-level `source` field, with clear default (cwd).

### G-5: No FR for diff capture mechanism

FR-015 says "capture diff and artifacts before sandbox cleanup" but doesn't specify: `git diff HEAD`? Against what baseline? What counts as an "artifact"?

---

## Inconsistencies

### I-1: Config format mismatch

- FR-085/086: `config.yaml`
- FR-095/098: `rein.toml`
- MEMORY.md: `rein.toml`

Pick one and use it everywhere. Recommend `rein.toml`.

### I-2: FR-060 scope ambiguity

Text says "on termination by Rein" but doesn't clarify behavior for `completed` runs where the agent already committed. Should state: only runs when Rein terminates the agent (context pressure or timeout), skipped on natural completion.

### I-3: FR-047 signal mapping includes non-MVP agents

"SIGINT for Codex, SIGTERM for Claude/Gemini" — MVP is Claude-only. Should state the general rule and note which applies for R1.

### I-4: FR-091 / FR-091a sub-ID numbering

Using sub-IDs mid-table breaks the clean numbering scheme. Consider making FR-091 (injection) and FR-091a (extraction) into two full IDs, renumbering downstream.

---

## Specification Tightness

### S-1: FR-036 "graceful kill" mechanism undefined

"Wait for current turn to complete" — what does this mean mechanistically? Monitor for the next complete NDJSON object then signal? Needs one sentence on the detection method.

### S-2: FR-038 "degraded mode" activation criteria

When does degraded mode activate? Only for agents that don't emit mid-turn tokens (Codex)? Also as a fallback when stream parsing fails mid-run? Needs explicit trigger conditions.

### S-3: Empty validation_commands scoring

FR-067 defines binary scoring but doesn't specify the result when `validation_commands` is empty. Score 1.0 (vacuous truth)? 0.0? null? FR-098 validates non-empty in strict mode, but warn/skip modes allow empty through.

### S-4: FR-092 "30-line heuristic" unspecified

`output_excerpt` uses a "30-line heuristic" — from where? Last 30 lines of stdout? First 30? Tail of the result text? Needs one sentence.

### S-5: FR-021 git identity source

No corresponding CLI flags or config keys to set author name/email. Hardcoded defaults? Configurable per-agent? Per-task?

---

## Platform Concern

S6 lists Windows as a target. FR-047's SIGTERM/SIGKILL are POSIX-only. No FR addresses Windows process termination (`TerminateProcess`, `CTRL_BREAK_EVENT`). Either add a Windows termination FR or explicitly scope R1 to POSIX.

---

## Minor Nits

- S1 repeats "comparison is a byproduct" from BRIEF.md — fine for standalone reading, could be one sentence.
- S7 "0% deviation" for token normalization is a tautology when summing the agent's own fields. The metric should measure *completeness* (no dropped events), not accuracy.

---

## Recommendations (Priority Order)

1. Add FR cluster for retry orchestration (G-1)
2. Reconcile config format to `rein.toml` everywhere (I-1)
3. Add FR for workspace source path (G-4)
4. Tighten FR-036/038 with one sentence each on mechanism (S-1, S-2)
5. Add FR for PROGRESS.md / DEFERRED.md seeding (G-2)
6. Specify empty-validation scoring behavior (S-3)
7. Address Windows process termination or scope R1 to POSIX
