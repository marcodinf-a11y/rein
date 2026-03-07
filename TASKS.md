# Rein — Task Definition Spec

Task files are **human-written, agent-read** — the developer authors them, Rein feeds them to agents. JSON is the supported task format. YAML support is planned as a future addition.

---

## TaskDefinition Dataclass

The `TaskDefinition` dataclass is the canonical in-memory representation, shared by all formats. When YAML support is added, the loader will detect the file extension and parse accordingly.

```python
@dataclass(frozen=True)
class WorkspaceConfig:
    """Controls how the sandbox directory is created."""
    type: str = "tempdir"   # "tempdir" | "worktree" | "copy"
    source: str = ""        # Path to existing repo/directory (required for worktree/copy)

@dataclass(frozen=True)
class TaskDefinition:
    id: str
    name: str
    prompt: str
    agent: str | None = None       # "claude", "codex", "gemini" — optional
    model: str | None = None       # Agent-specific model identifier — optional
    effort: str | None = None      # "low", "medium", "high" — optional
    workspace: WorkspaceConfig = field(default_factory=WorkspaceConfig)
    setup_commands: list[str] = field(default_factory=list)
    validation_commands: list[str] = field(default_factory=list)
    files: dict[str, str] = field(default_factory=dict)
    token_budget: int = 70_000
    timeout_seconds: int = 300
    tags: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
```

### Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | `str` | yes | — | Unique task identifier |
| `name` | `str` | yes | — | Human-readable task name |
| `prompt` | `str` | yes | — | Prompt sent to the agent |
| `agent` | `str \| None` | no | `None` | Agent: `claude`, `codex`, `gemini`. CLI `--agent` provides default. |
| `model` | `str \| None` | no | `None` | Model identifier passed to agent CLI (e.g. `opus`, `flash`). CLI `--model` provides default. |
| `effort` | `str \| None` | no | `None` | Reasoning effort: `low`, `medium`, `high`. CLI `--effort` provides default. |
| `workspace` | `WorkspaceConfig` | no | `{type: "tempdir"}` | Sandbox creation strategy (see [Workspace Types](#workspace-types)) |
| `setup_commands` | `list[str]` | no | `[]` | Commands to run before the agent starts |
| `validation_commands` | `list[str]` | no | `[]` | Commands to run after the agent finishes |
| `files` | `dict[str, str]` | no | `{}` | Filename to content mapping to seed in sandbox |
| `token_budget` | `int` | no | `70000` | Token budget for the agent run |
| `timeout_seconds` | `int` | no | `300` | Wall-clock timeout in seconds. On expiry, Rein applies the [Subprocess Termination Procedure](ARCHITECTURE.md#subprocess-termination-procedure) (immediate mode): `SIGTERM`/`SIGINT`, 5-second grace period, then `SIGKILL`. All complete NDJSON/JSONL lines are preserved as partial output. Report sets `termination_reason=timed_out`. |
| `tags` | `list[str]` | no | `[]` | For filtering and categorization |
| `metadata` | `dict[str, Any]` | no | `{}` | Arbitrary metadata |

### Agent, Model, and Effort Resolution

CLI flags (`--agent`, `--model`, `--effort`) provide **defaults** for tasks that do not specify their own values. Task-level fields take precedence over CLI flags, enabling composition pipelines where different tasks target different agents/models/effort levels.

| Task field | CLI flag | Result |
|---|---|---|
| set | set | Task value wins |
| set | absent | Task value |
| absent | set | CLI value |
| absent | absent | Agent default (model/effort) or error (agent) |

An agent must be resolvable for every task — from the task's `agent` field or the CLI `--agent` flag. If neither provides an agent, Rein errors. Model and effort are always optional; agents use their own defaults when not specified.

### WorkspaceConfig Fields

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `type` | `str` | no | `"tempdir"` | `"tempdir"`, `"worktree"`, or `"copy"` |
| `source` | `str` | conditional | `""` | Path to existing repo or directory. Required when `type` is `"worktree"` or `"copy"`. Ignored for `"tempdir"`. |

---

## Workspace Types

The `workspace` field controls how the sandbox directory is created. This determines whether the agent works in a fresh empty directory (greenfield) or against an existing codebase (brownfield).

### `tempdir` (default)

Creates an empty temporary directory. This is the current behavior and the default when `workspace` is omitted.

- No `source` required
- Seed files from `files` are written into the empty directory
- `setup_commands` run after seeding (e.g. `git init`, `python3 -m venv .venv`)
- Diff baseline: initial commit after setup, or no baseline if no git repo is initialized
- Cleanup: directory is deleted after artifacts are captured

Best for: greenfield tasks, algorithm challenges, self-contained modules.

### `worktree`

Creates a git worktree from an existing repository. The agent works on a detached branch — the original working tree is untouched.

- `source` is required and must point to a git repository
- Rein runs `git worktree add <sandbox_path> -b rein/<task_id> HEAD` from the source repo
- Seed files from `files` are written into the worktree (overwriting existing files or adding new ones)
- `setup_commands` run after seeding
- Diff baseline: the commit at worktree creation time (`HEAD` of source at invocation)
- Cleanup: `git worktree remove <sandbox_path>` and `git branch -D rein/<task_id>`

Best for: brownfield tasks against git repositories — refactoring, bug fixes, feature additions. The agent has access to the full project structure, existing tests, and dependencies.

### `copy`

Copies an existing directory into a temporary directory. The agent works on the copy — the original is untouched.

- `source` is required and must point to an existing directory
- Rein copies the source tree into a fresh temp directory (respecting `.gitignore` if present)
- If the source is a git repo, the copy includes `.git` — diff baseline is the current commit
- If the source is not a git repo, Rein runs `git init` and creates an initial commit to establish a diff baseline
- Seed files from `files` are written into the copy (overwriting or adding)
- `setup_commands` run after seeding
- Cleanup: temp directory is deleted after artifacts are captured

Best for: non-git projects, or when you need full isolation (e.g. destructive setup commands, testing different approaches against the same snapshot).

---

## JSON Format

### Why JSON

**Generation reliability**: JSON has the highest structured output reliability across all LLMs. StructEval benchmarks show GPT-4o at 99.36% accuracy for JSON generation. All three providers (Anthropic, OpenAI, Google) support only JSON for constrained decoding. Source: [StructEval (arXiv)](https://arxiv.org/html/2505.20139v1).

**Training data prevalence**: JSON is vastly more prevalent in LLM training corpora — it appears embedded in virtually every programming language. This training data advantage translates to more reliable parsing and understanding, especially in smaller models.

**Strict syntax**: JSON has no implicit type coercion, no indentation-sensitive parsing, and one unambiguous representation for any given structure. A malformed JSON file fails loudly at parse time — there are no silent misinterpretations.

**Tooling**: JSON Schema validation is a mature ecosystem. Every language has robust JSON parsers. IDE support (syntax highlighting, formatting, linting) is universal.

**Known JSON limitations (mitigated)**:
- No comments: mitigated by using `metadata` field for annotations, or a companion README per task directory
- Multiline strings: mitigated by `\n` escapes — less readable but unambiguous
- Verbosity: ~24% more tokens than YAML — a cost tradeoff, not a correctness issue

### JSON Schema

```json
{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "required": ["id", "name", "prompt"],
    "properties": {
        "id":                  { "type": "string", "description": "Unique task identifier" },
        "name":                { "type": "string", "description": "Human-readable task name" },
        "prompt":              { "type": "string", "description": "Prompt sent to the agent" },
        "agent":               { "type": "string", "enum": ["claude", "codex", "gemini"], "description": "Agent for this task (CLI --agent provides default)" },
        "model":               { "type": "string", "description": "Model identifier passed to agent CLI (CLI --model provides default)" },
        "effort":              { "type": "string", "enum": ["low", "medium", "high"], "description": "Reasoning effort level (CLI --effort provides default)" },
        "workspace": {
            "type": "object",
            "description": "Sandbox creation strategy",
            "properties": {
                "type":   { "type": "string", "enum": ["tempdir", "worktree", "copy"], "default": "tempdir" },
                "source": { "type": "string", "description": "Path to existing repo/directory (required for worktree/copy)", "default": "" }
            },
            "additionalProperties": false,
            "default": { "type": "tempdir" }
        },
        "setup_commands":      { "type": "array", "items": { "type": "string" }, "default": [] },
        "validation_commands": { "type": "array", "items": { "type": "string" }, "default": [] },
        "files":               { "type": "object", "additionalProperties": { "type": "string" }, "default": {} },
        "token_budget":        { "type": "integer", "default": 70000 },
        "timeout_seconds":     { "type": "integer", "default": 300, "description": "Wall-clock timeout in seconds. On expiry: SIGTERM/SIGINT → 5s grace → SIGKILL. Partial NDJSON/JSONL output preserved. Sets termination_reason=timed_out." },
        "tags":                { "type": "array", "items": { "type": "string" }, "default": [] },
        "metadata":            { "type": "object", "default": {} }
    },
    "if": {
        "properties": { "workspace": { "properties": { "type": { "enum": ["worktree", "copy"] } } } }
    },
    "then": {
        "properties": { "workspace": { "required": ["source"] } }
    },
    "additionalProperties": false
}
```

### JSON Examples

**FizzBuzz**:

```json
{
    "id": "fizzbuzz-001",
    "name": "FizzBuzz implementation",
    "prompt": "Create a Python file called fizzbuzz.py that implements FizzBuzz for numbers 1-100.\nPrint \"Fizz\" for multiples of 3, \"Buzz\" for multiples of 5, \"FizzBuzz\" for both.\nInclude a main() function and if __name__ == \"__main__\" guard.",
    "setup_commands": [
        "git init",
        "python3 -m venv .venv"
    ],
    "validation_commands": [
        "python3 fizzbuzz.py | head -15",
        "python3 -c \"import fizzbuzz; fizzbuzz.main()\""
    ],
    "files": {
        "requirements.txt": ""
    },
    "token_budget": 70000,
    "timeout_seconds": 120,
    "tags": ["easy", "python", "basics"],
    "metadata": {
        "expected_files": ["fizzbuzz.py"],
        "difficulty": "easy"
    }
}
```

**REST API**:

```json
{
    "id": "rest-api-001",
    "name": "Flask REST API with tests",
    "prompt": "Create a Flask REST API with:\n- GET /items - list all items\n- POST /items - create an item (json body: {\"name\": \"...\", \"price\": ...})\n- GET /items/<id> - get item by ID\n- DELETE /items/<id> - delete item\nUse an in-memory dict for storage. Include pytest tests in test_app.py.",
    "setup_commands": [
        "git init",
        "python3 -m venv .venv",
        ".venv/bin/pip install flask pytest"
    ],
    "validation_commands": [
        ".venv/bin/python -c 'import app'",
        ".venv/bin/pytest test_app.py -v"
    ],
    "files": {
        "requirements.txt": "flask\npytest\n"
    },
    "token_budget": 70000,
    "timeout_seconds": 300,
    "tags": ["medium", "python", "api", "testing"],
    "metadata": {
        "expected_files": ["app.py", "test_app.py"],
        "difficulty": "medium"
    }
}
```

**Brownfield — Refactor existing module** (worktree):

```json
{
    "id": "refactor-auth-001",
    "name": "Refactor auth module to use JWT",
    "prompt": "Refactor src/auth.py to use PyJWT instead of the custom token implementation.\nKeep the same public API (create_token, verify_token, refresh_token).\nUpdate tests in tests/test_auth.py to cover the new implementation.\nRun the full test suite to verify nothing else breaks.",
    "workspace": {
        "type": "worktree",
        "source": "/home/dev/projects/myapp"
    },
    "setup_commands": [
        "pip install pyjwt"
    ],
    "validation_commands": [
        "python -c 'from src.auth import create_token, verify_token, refresh_token'",
        "pytest tests/test_auth.py -v",
        "pytest tests/ -v --timeout=60"
    ],
    "token_budget": 70000,
    "timeout_seconds": 300,
    "tags": ["medium", "python", "refactor", "auth"],
    "metadata": {
        "target_files": ["src/auth.py", "tests/test_auth.py"],
        "difficulty": "medium"
    }
}
```

**Brownfield — Fix bug in existing project** (copy):

```json
{
    "id": "fix-parser-001",
    "name": "Fix CSV parser edge case",
    "prompt": "The CSV parser in lib/csv_parser.py crashes on quoted fields containing newlines.\nFix the bug and add a regression test in tests/test_csv_parser.py.\nThe failing input is: '\"hello\\nworld\",42'",
    "workspace": {
        "type": "copy",
        "source": "/home/dev/projects/data-tools"
    },
    "files": {
        "tests/fixtures/newline_in_quotes.csv": "name,value\n\"hello\nworld\",42\n"
    },
    "validation_commands": [
        "pytest tests/test_csv_parser.py -v -k newline"
    ],
    "token_budget": 50000,
    "timeout_seconds": 180,
    "tags": ["easy", "python", "bugfix"],
    "metadata": {
        "target_files": ["lib/csv_parser.py", "tests/test_csv_parser.py"],
        "difficulty": "easy"
    }
}
```

**Workflow composition** — tasks targeting different agents and models:

```json
{
    "id": "review-auth-001",
    "name": "Review auth refactoring",
    "agent": "claude",
    "model": "opus",
    "effort": "high",
    "prompt": "Review the changes in src/auth.py for security issues, edge cases,\nand API contract violations. Check that all tests in tests/test_auth.py\ncover the new JWT implementation. Report findings as inline comments.",
    "workspace": {
        "type": "worktree",
        "source": "/home/dev/projects/myapp"
    },
    "validation_commands": [
        "pytest tests/test_auth.py -v"
    ],
    "token_budget": 50000,
    "timeout_seconds": 180,
    "tags": ["review", "auth", "security"]
}
```

In a composition pipeline, a directory of tasks can target different agents:

```
tasks/pipeline/
    01_plan.json         → agent: claude, model: opus, effort: high
    02_implement.json    → agent: gemini, model: flash, effort: medium
    03_review.json       → agent: claude, model: sonnet
```

Run with: `rein run -t tasks/pipeline/`

Each task dispatches to its specified agent. Tasks without an `agent` field require `--agent` on the CLI.

---

## YAML Format [Planned]

> **Status: Planned.** YAML support will be added in a future release. The sections below describe the intended design. All examples use future tense.

YAML support will be added as an alternative task format. The `TaskDefinition` dataclass is format-agnostic — adding YAML will require only a loader that detects file extension.

### Why YAML

**Comprehension accuracy**: The ImprovingAgents benchmark (1,000 questions per format, nested data retrieval across 3 models) found YAML outperformed JSON on 2 of 3 models — by 8-12 percentage points on GPT-5 Nano and Gemini 2.5 Flash Lite. Only Llama 3.2 3B slightly preferred JSON (+3.6pp). Source: [ImprovingAgents](https://www.improvingagents.com/blog/best-nested-data-format/).

**Multiline prompts**: The `prompt` field is the core of every task. YAML will handle this naturally with `|` block scalars. In JSON, the same content becomes an unreadable string with `\n` escapes.

**Token efficiency**: YAML uses ~24% fewer tokens than readable JSON. For task definitions that are sent as agent input, this will directly reduce cost and leave more context window for the agent's work. Sources: [Sean Ryan](https://medium.com/@mr.sean.ryan/reduce-llm-costs-and-increase-speed-consider-switching-to-yaml-instead-of-json-62af2f7a37c0), [Piotr Sikora](https://www.piotr-sikora.com/blog/2025-12-05-toon-tron-csv-yaml-json-format-comparison).

**Ecosystem alignment**: Every major agent framework uses YAML for human-edited config: CrewAI, Google ADK, Julep AI, Aider. Claude Code and Cursor use Markdown with YAML frontmatter.

**Known YAML risks (mitigated)**:
- Indentation errors: will be mitigated by schema validation at load time
- Implicit type coercion (`yes` -> `true`, `3.10` -> `3.1`): will be mitigated by quoting string values and validating types
- Parsing bugs in LLM-generated YAML: not applicable — task files are human-written

### YAML Schema [Planned]

The YAML schema will map directly to the same `TaskDefinition` dataclass:

```yaml
id: string                        # Unique task identifier
name: string                      # Human-readable task name
prompt: string                    # Prompt sent to the agent (multiline with |)
agent: string                     # Optional: claude, codex, gemini
model: string                     # Optional: agent-specific model identifier
effort: string                    # Optional: low, medium, high
workspace:                        # Sandbox creation strategy
  type: string                    # "tempdir" (default) | "worktree" | "copy"
  source: string                  # Path to existing repo/directory
setup_commands: list[string]      # Commands to run before the agent starts
validation_commands: list[string] # Commands to run after the agent finishes
files: dict[string, string]       # filename -> content to seed in sandbox
token_budget: int                 # Default: 70000
timeout_seconds: int              # Default: 300
tags: list[string]                # For filtering/categorization
metadata: dict[string, any]       # Arbitrary metadata
```

### YAML Examples [Planned]

**FizzBuzz** — this task definition will look like:

```yaml
# tasks/example_fizzbuzz.yaml
id: fizzbuzz-001
name: "FizzBuzz implementation"
prompt: |
  Create a Python file called fizzbuzz.py that implements FizzBuzz for numbers 1-100.
  Print "Fizz" for multiples of 3, "Buzz" for multiples of 5, "FizzBuzz" for both.
  Include a main() function and if __name__ == "__main__" guard.

setup_commands:
  - "git init"
  - "python3 -m venv .venv"

validation_commands:
  - "python3 fizzbuzz.py | head -15"
  - "python3 -c \"import fizzbuzz; fizzbuzz.main()\""

files:
  requirements.txt: ""

token_budget: 70000
timeout_seconds: 120
tags: ["easy", "python", "basics"]
metadata:
  expected_files: ["fizzbuzz.py"]
  difficulty: "easy"
```

**REST API** — this task definition will look like:

```yaml
id: rest-api-001
name: "Flask REST API with tests"
prompt: |
  Create a Flask REST API with:
  - GET /items - list all items
  - POST /items - create an item (json body: {"name": "...", "price": ...})
  - GET /items/<id> - get item by ID
  - DELETE /items/<id> - delete item
  Use an in-memory dict for storage. Include pytest tests in test_app.py.

setup_commands:
  - "git init"
  - "python3 -m venv .venv"
  - ".venv/bin/pip install flask pytest"

validation_commands:
  - ".venv/bin/python -c 'import app'"
  - ".venv/bin/pytest test_app.py -v"

files:
  requirements.txt: |
    flask
    pytest

token_budget: 70000
timeout_seconds: 300
tags: ["medium", "python", "api", "testing"]
metadata:
  expected_files: ["app.py", "test_app.py"]
  difficulty: "medium"
```

**Brownfield — Refactor existing module** (worktree) — this task definition will look like:

```yaml
id: refactor-auth-001
name: "Refactor auth module to use JWT"
prompt: |
  Refactor src/auth.py to use PyJWT instead of the custom token implementation.
  Keep the same public API (create_token, verify_token, refresh_token).
  Update tests in tests/test_auth.py to cover the new implementation.
  Run the full test suite to verify nothing else breaks.

workspace:
  type: worktree
  source: /home/dev/projects/myapp

setup_commands:
  - "pip install pyjwt"

validation_commands:
  - "python -c 'from src.auth import create_token, verify_token, refresh_token'"
  - "pytest tests/test_auth.py -v"
  - "pytest tests/ -v --timeout=60"

token_budget: 70000
timeout_seconds: 300
tags: ["medium", "python", "refactor", "auth"]
metadata:
  target_files: ["src/auth.py", "tests/test_auth.py"]
  difficulty: "medium"
```

---

## Validation Patterns

After the agent completes, Rein runs each entry in `validation_commands` inside the sandbox. Each command is executed as a shell subprocess.

Scoring is binary: all commands exit 0 -> score 1.0, any non-zero -> score 0.0. Stdout/stderr are captured in the report. See [REPORTS.md](REPORTS.md) for scoring details.

### Common Patterns (JSON)

```json
{ "validation_commands": ["test -f output.py"] }
```

```json
{ "validation_commands": ["python3 main.py"] }
```

```json
{ "validation_commands": ["pytest tests/ -v"] }
```

```json
{ "validation_commands": ["python3 -c 'import mymodule'", "pytest tests/ -v", "ruff check ."] }
```

### Common Patterns (YAML) [Planned]

```yaml
validation_commands:
  - "test -f output.py"
```

```yaml
validation_commands:
  - "python3 main.py"
```

```yaml
validation_commands:
  - "pytest tests/ -v"
```

```yaml
validation_commands:
  - "python3 -c 'import mymodule'"
  - "pytest tests/ -v"
  - "ruff check ."
```

---

## File Seeding

The `files` field seeds the sandbox before the agent runs. Keys are relative paths, values are file contents. Directories are created automatically.

### JSON Example

```json
{
    "files": {
        "requirements.txt": "",
        "config.json": "{\"debug\": true, \"port\": 8080}",
        "src/utils.py": "def helper():\n    pass\n"
    }
}
```

### YAML Example [Planned]

```yaml
files:
  requirements.txt: ""
  config.json: |
    {"debug": true, "port": 8080}
  src/utils.py: |
    def helper():
        pass
```

---

## Task Directory Convention

Store tasks in `tasks/`:

```
tasks/
    example_fizzbuzz.json
    rest_api.json
    refactor_auth.json
```

Run a single task or an entire directory:

```bash
rein run -t tasks/example_fizzbuzz.json -a claude
rein run -t tasks/ -a claude
```

Rein accepts `.json` files. When scanning a directory, it loads all `.json` files. YAML support (`.yaml`) will be added in a future release.

---

## Token Budget

The `token_budget` field defaults to `70,000` tokens per task. Overrides are available per-task (in the task definition) and per-CLI run.

Token budget configuration and threshold details are documented in [TOKENS.md](TOKENS.md).
