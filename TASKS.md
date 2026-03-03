# Agentic Harness — Task Definition Spec

Task files are **human-written, agent-read** — the developer authors them, the harness feeds them to agents. JSON is the MVP task format. YAML support is planned as a future addition.

---

## TaskDefinition Dataclass

The `TaskDefinition` dataclass is the canonical in-memory representation, shared by all formats. When YAML support is added, the loader will detect the file extension and parse accordingly.

```python
@dataclass(frozen=True)
class TaskDefinition:
    id: str
    name: str
    prompt: str
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
| `setup_commands` | `list[str]` | no | `[]` | Commands to run before the agent starts |
| `validation_commands` | `list[str]` | no | `[]` | Commands to run after the agent finishes |
| `files` | `dict[str, str]` | no | `{}` | Filename to content mapping to seed in sandbox |
| `token_budget` | `int` | no | `70000` | Token budget for the agent run |
| `timeout_seconds` | `int` | no | `300` | Wall-clock timeout in seconds |
| `tags` | `list[str]` | no | `[]` | For filtering and categorization |
| `metadata` | `dict[str, Any]` | no | `{}` | Arbitrary metadata |

---

## JSON Format (MVP)

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
        "setup_commands":      { "type": "array", "items": { "type": "string" }, "default": [] },
        "validation_commands": { "type": "array", "items": { "type": "string" }, "default": [] },
        "files":               { "type": "object", "additionalProperties": { "type": "string" }, "default": {} },
        "token_budget":        { "type": "integer", "default": 70000 },
        "timeout_seconds":     { "type": "integer", "default": 300 },
        "tags":                { "type": "array", "items": { "type": "string" }, "default": [] },
        "metadata":            { "type": "object", "default": {} }
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

---

## Validation Patterns

After the agent completes, the harness runs each entry in `validation_commands` inside the sandbox. Each command is executed as a shell subprocess.

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
harness run -t tasks/example_fizzbuzz.json -a claude
harness run -t tasks/ -a claude
```

The MVP harness accepts `.json` files. When scanning a directory, it loads all `.json` files. YAML support (`.yaml`) will be added in a future release.

---

## Token Budget

The `token_budget` field defaults to `70,000` tokens per task. Overrides are available per-task (in the task definition) and per-CLI run.

Token budget configuration and threshold details are documented in [TOKENS.md](TOKENS.md).
