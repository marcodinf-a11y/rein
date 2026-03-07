# Toolshed and MCP: Stripe's 400+ Tool Ecosystem

## Executive Summary

Stripe's "Toolshed" is a centralized internal MCP (Model Context Protocol) server hosting ~500 tools that span internal systems and external SaaS platforms. It serves as the primary interface between Minions agents and Stripe's developer infrastructure. The critical architectural insight is not the tool count but how tools are curated: the orchestrator deterministically selects a surgical subset of ~15 tools per task, solving the context window problem that would make 500 tool definitions unmanageable. Toolshed is MCP-compliant, not a proprietary protocol.

---

## Research Question 1: What is Toolshed?

### Confirmed (from Stripe blog posts)

- Toolshed is a **single centralized internal MCP server** hosting tools spanning internal systems and external SaaS platforms.
- The blog posts describe it as "a central internal MCP server called Toolshed" (Part 2).
- Tool count is stated as "nearly 500" in some references and "over 400" in others -- the number has likely grown between the Part 1 and Part 2 publications.
- It uses MCP as its protocol, described as providing "a common language for networkable LLM function calling."

### Inferred

- Toolshed is likely a single logical server that may be backed by multiple service integrations. MCP servers can proxy to multiple backends while presenting a unified tool interface to clients. The phrase "spanning internal systems and SaaS platforms" suggests it aggregates access to many disparate services behind one MCP endpoint.
- Given Stripe's monorepo architecture and their emphasis on developer tooling consistency, Toolshed likely runs as an internal service accessible within Stripe's network, not as a sidecar per devbox.

### Unknown

- Whether Toolshed is a single process or a federated gateway aggregating multiple MCP servers.
- The internal implementation language and deployment model (standalone service vs. per-devbox instance).
- Whether Toolshed predates MCP and was retrofitted, or was built MCP-native from the start.

---

## Research Question 2: Tool Categories

### Confirmed (from Stripe blog posts)

The blog posts explicitly mention these tool capabilities:

| Category | Examples Mentioned | Source |
|---|---|---|
| **Code intelligence** | Sourcegraph code search | Part 1, Part 2 |
| **Internal documentation** | Fetching internal docs | Part 1, Part 2 |
| **Ticket/issue tracking** | Pulling Jira ticket details | Part 1, Part 2 |
| **Build/CI status** | Build status information | Part 2 |
| **External SaaS** | Unspecified SaaS platform integrations | Part 2 |

### Inferred

Based on the stated capabilities and Stripe's known infrastructure, the ~500 tools likely include:

- **Code manipulation**: File read/write/edit, git operations (though some git operations are handled deterministically by the Blueprint, not as MCP tools)
- **Code search and navigation**: Sourcegraph search, symbol lookup, cross-reference queries
- **Documentation access**: Internal wiki/docs retrieval, API documentation lookup
- **Issue tracking**: Jira/internal tracker read/write, ticket status updates
- **CI/CD interaction**: Build status queries, test result retrieval, lint result parsing
- **Internal service APIs**: Stripe-specific service queries (likely read-only for safety)
- **Monitoring/observability**: Log search, error aggregation queries
- **External SaaS connectors**: Slack integration, possibly PagerDuty, GitHub/internal Git hosting

The large count (~500) suggests many tools are narrow, service-specific wrappers rather than broad general-purpose tools. At Stripe's scale with many internal microservices, having dedicated tools per service domain would quickly reach hundreds.

### Unknown

- The exact distribution across categories.
- Whether tools include write operations to production systems or are read-only.
- Which external SaaS platforms are integrated.

---

## Research Question 3: Tool Selection Per-Task

### Confirmed (from Stripe blog posts)

This is one of the most well-documented aspects:

- **The orchestrator performs deterministic prefetching**: "scanning the prompt for links and keywords, finding relevant documentation and tickets, and curating a surgical subset of approximately 15 relevant tools."
- **Not all tools are exposed**: "Providing an LLM with 400 tools wastes valuable tokens as it attempts to determine which to use."
- **Pre-hydration before agent loop**: "Relevant MCP tools are deterministically run over likely-looking links before a minion run even starts to hydrate context."
- **Blueprint-driven selection**: The Blueprint pattern specifies which tools are relevant for each task type.

### Inferred

The tool selection mechanism has two distinct phases:

1. **Static selection by Blueprint**: Each Blueprint (task type template) specifies a base set of tools relevant to that category of work. A "fix flaky test" Blueprint would include test-related tools; a "documentation update" Blueprint would include doc-related tools.

2. **Dynamic enrichment by orchestrator**: Before the agent loop begins, the orchestrator scans the task description/prompt for links, keywords, and references, then deterministically calls relevant MCP tools to pre-fetch context (ticket details, docs, code references). This is not the agent choosing tools -- it is rein choosing tools for the agent.

This two-phase approach means the agent sees ~15 tools and pre-fetched context, not 500 tool definitions. The agent can still call tools during its loop, but from the curated subset.

### Unknown

- Whether the agent can request additional tools not in the initial subset.
- The exact mechanism for Blueprint-to-tool mapping (config file, code, heuristic).
- Whether the ~15 number is a hard limit or varies by task type.

---

## Research Question 4: Context Window Management

### Confirmed

- Stripe explicitly acknowledges the context problem: providing 400+ tool definitions "wastes valuable tokens."
- Their solution is curated subsets (~15 tools) rather than lazy loading or deferred tool schemas.
- Context is pre-hydrated with actual content (ticket details, docs, code) before the agent loop starts, so the agent begins with rich, relevant context rather than needing to discover and fetch it.

### Analysis: Stripe's Approach vs. Industry Approaches

The MCP ecosystem has developed several approaches to the "too many tools" problem:

| Approach | How It Works | Who Uses It |
|---|---|---|
| **Curated subsets** | Orchestrator selects ~15 tools per task | Stripe Toolshed |
| **Lazy/deferred loading** | Load tool definitions on-demand when needed | Claude Code (ToolSearch), OpenCode |
| **Schema partitioning** | Load tool names first, fetch full schemas only when called | OpenMCP proposal |
| **Pagination** | Server returns tools in pages via cursor-based pagination | MCP spec (tools/list) |

Stripe chose the most aggressive approach: pre-selecting tools before the agent even starts. This is possible because their tasks are well-categorized by Blueprints, making tool relevance highly predictable.

**Claude Code's ToolSearch** (the mechanism used in this very agent session) is a deferred loading system: when MCP tool descriptions would consume >10% of the context window, it builds a lightweight index and loads full tool definitions on-demand. This achieves ~95% reduction in initial context consumption (from ~108K tokens to ~5K).

**MCP specification** supports pagination via cursor-based `tools/list` responses (spec version 2025-06-18), but this is a transport-level optimization, not a context-level one. Even with pagination, all tools would still be loaded into context if the client requests them all.

### Inferred

Stripe's approach works because of tight coupling between Blueprints and tool sets. For a system without well-typed task categories, lazy loading (Claude Code style) or dynamic tool search would be more appropriate. The tradeoff is:

- **Stripe's approach**: Higher upfront engineering (must map tools to task types), but zero wasted context tokens.
- **Lazy loading**: Lower setup cost, works with arbitrary task types, but adds latency and requires the agent to describe what it needs.

### Unknown

- Whether Stripe has experimented with lazy/deferred tool loading.
- The exact token cost of ~15 tool definitions (likely 5K-8K tokens, very manageable).
- Whether they compress or truncate tool descriptions.

---

## Research Question 5: Relationship to Internal APIs

### Confirmed

- Toolshed tools "span internal systems and SaaS platforms."
- Tools provide access to "internal documentation, ticket details, build statuses, and code intelligence via Sourcegraph search."
- The devbox environment is "isolated from production and the internet."

### Inferred

- Tools are **thin wrappers around internal services**, following the MCP pattern of exposing service capabilities as callable functions with structured inputs/outputs.
- Given that devboxes are isolated from production and the internet, Toolshed must be reachable from within the devbox network while production services are not. This suggests Toolshed acts as a **controlled gateway** -- the agent can call Toolshed tools (which may read from internal services), but cannot directly access those services.
- This is a security architecture choice: the agent's blast radius is limited to what Toolshed exposes, not the full internal API surface.

### Unknown

- Whether Toolshed tools have write access to any internal systems (e.g., updating Jira tickets, posting Slack messages) or are read-only.
- Whether there are rate limits or permission scoping per tool.
- The authentication model between devbox and Toolshed.

---

## Research Question 6: Tool Versioning, Testing, and Maintenance

### Confirmed

Nothing is explicitly stated in the blog posts about tool versioning, testing, or maintenance at scale.

### Inferred

At ~500 tools, Stripe must have:

- **A tool definition standard**: Likely a schema or template that tool authors follow, given the centralized Toolshed model. MCP provides this naturally via the tool definition format (name, description, input schema).
- **Centralized ownership**: A single Toolshed server implies a team or platform group owns the tool registry, rather than distributed ownership across service teams.
- **Testing via the Blueprint pipeline**: Tools are exercised by every Minion run, providing continuous integration testing of tool reliability. Flaky or broken tools would immediately surface as Minion failures.
- **Versioning via the monorepo**: Stripe operates a monorepo. Tool definitions likely live in the monorepo alongside the services they wrap, meaning standard code review and CI apply.

### Unknown

- Whether there is a formal tool review/approval process for new tools.
- How deprecated tools are handled.
- Whether tool quality metrics (call success rate, latency, etc.) are tracked.
- The ownership model -- platform team vs. service team per tool.

---

## Research Question 7: MCP Compliance

### Confirmed

- Toolshed is explicitly described as an "MCP server."
- MCP is described as providing "a common language for networkable LLM function calling."
- Stripe's Minions are built on Goose, which natively supports MCP as its extension protocol.

### Inferred

- Toolshed is **MCP-compliant**, not a proprietary protocol. This is a natural choice given that:
  - Goose (the base agent) uses MCP natively for all extensions.
  - MCP was created by Anthropic and published as an open spec in late 2024.
  - Building a proprietary protocol would mean forking Goose's extension system, adding unnecessary maintenance burden.
- The MCP protocol supports all the communication patterns Toolshed needs: tool discovery (`tools/list`), tool execution (`tools/call`), and both STDIO and HTTP (Streamable HTTP) transports.

### Unknown

- Whether Stripe uses STDIO or HTTP transport for Toolshed communication.
- Whether they have extended MCP with proprietary additions (custom methods, metadata).
- The MCP protocol version they target.

---

## Goose Extension System: The Foundation

Since Toolshed runs atop a Goose fork, understanding Goose's extension architecture is relevant:

### Goose Extension Types

| Type | Transport | Description |
|---|---|---|
| **Builtin** | In-process (Rust) | Compiled into the Goose binary, implement McpClientTrait directly |
| **STDIO** | stdin/stdout pipes | External processes communicating via MCP over STDIO |
| **SSE** | HTTP + Server-Sent Events | Remote MCP servers via HTTP with SSE streaming |
| **Streamable HTTP** | HTTP POST + optional SSE | Modern MCP transport (spec 2025-03-26+) |

Goose acts as both an MCP host and manages multiple MCP clients. Each extension runs as a separate MCP server. All built-in extensions are MCP servers, interoperable with other MCP clients (Claude Desktop, Cursor, etc.).

Stripe's Toolshed likely connects to Goose as either an SSE or Streamable HTTP extension, since it is described as a centralized server rather than a per-process sidecar.

---

## Comparison: Toolshed vs. Other Tool Systems

| Dimension | Stripe Toolshed | Claude Code MCP | Codex CLI | Goose (vanilla) |
|---|---|---|---|---|
| **Protocol** | MCP | MCP | MCP | MCP |
| **Tool count** | ~500 | Varies (external servers) | Varies (external servers) | Varies (extensions) |
| **Selection strategy** | Deterministic curation (~15/task) | Lazy loading (ToolSearch) | Full load | Full load |
| **Context optimization** | Pre-selection eliminates waste | Deferred schemas (95% reduction) | None documented | None documented |
| **Server topology** | Single centralized server | Multiple per-config servers | Multiple per-config servers | Multiple extensions |
| **Tool ownership** | Internal platform team | User-configured | User-configured | User-configured |
| **Pre-hydration** | Yes (before agent loop) | No | No | No |

---

## Key Architectural Insights

### 1. The Real Innovation is Selection, Not Scale

Having 500 tools is not novel -- the MCP ecosystem has 3,000+ servers available. Stripe's contribution is the **deterministic tool selection** pattern: the orchestrator, not the agent, decides which tools are relevant. This is cheaper, faster, and more reliable than having the agent discover tools dynamically.

### 2. Pre-Hydration Eliminates Agent Discovery Loops

By running relevant MCP tools before the agent loop starts (fetching ticket details, docs, code context), Stripe eliminates the common "agent spends 5 turns figuring out what to look at" problem. The agent begins its first turn with all relevant context already in its prompt.

### 3. Centralized Tool Server Enables Security Boundaries

Toolshed acts as a controlled gateway between the isolated devbox and Stripe's internal services. The agent can only access what Toolshed exposes, not the full internal API surface. This is a deliberate security architecture choice.

### 4. MCP as Standard Eliminates Protocol Lock-in

By using standard MCP rather than a proprietary protocol, Stripe can leverage the Goose ecosystem, swap underlying models, and potentially contribute tools back to the open-source ecosystem. The protocol choice was likely inherited from Goose rather than a conscious strategic decision, but it has significant benefits.

### 5. Tool Proliferation is Managed, Not Prevented

Rather than fighting tool proliferation (trying to keep the count low), Stripe embraces it and solves the downstream problem (context window pressure) through selection. This is pragmatic: in a large org with many internal services, having a tool per service is natural and maintainable.

---

## Sources

### Primary Sources (Stripe Official)
- [Minions: Stripe's one-shot, end-to-end coding agents (Part 1)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)
- [Minions: Stripe's one-shot, end-to-end coding agents -- Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [Stripe MCP Documentation](https://docs.stripe.com/mcp)

### Secondary Sources (Analysis and Commentary)
- [Ry Walker Research: Stripe Minions](https://rywalker.com/research/stripe-minions)
- [What Stripe's Minions Get Right About Coding Agents](https://www.mrphilgames.com/blog/what-stripes-minions-get-right-about-coding-agents)
- [Deconstructing Stripe's 'Minions': One-Shot Agents at Scale (SitePoint)](https://www.sitepoint.com/stripe-minions-architecture-explained/)
- [Engineering.fyi: Minions Part 1](https://www.engineering.fyi/article/minions-stripe-s-one-shot-end-to-end-coding-agents)
- [Engineering.fyi: Minions Part 2](https://www.engineering.fyi/article/minions-stripe-s-one-shot-end-to-end-coding-agents-part-2)
- [The Emerging "Harness Engineering" Playbook](https://www.ignorance.ai/p/the-emerging-harness-engineering)

### MCP Protocol and Ecosystem
- [MCP Specification: Tools (2025-06-18)](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)
- [MCP Pagination Specification](https://modelcontextprotocol.info/specification/draft/server/utilities/pagination/)
- [Claude Code Lazy Loading for MCP Tools](https://github.com/anthropics/claude-code/issues/7336)
- [OpenMCP: Lazy Loading Input Schemas](https://www.open-mcp.org/blog/lazy-loading-input-schemas)
- [Hierarchical Tool Management for MCP Discussion](https://github.com/orgs/modelcontextprotocol/discussions/532)

### Goose Agent Architecture
- [Goose Architecture Documentation](https://block.github.io/goose/docs/goose-architecture/)
- [Goose GitHub Repository](https://github.com/block/goose)
- [Goose Extension Types (DeepWiki)](https://deepwiki.com/block/goose/5.3-extension-types-and-configuration)
- [Deep Dive into Goose Extension System and MCP (DEV Community)](https://dev.to/lymah/deep-dive-into-gooses-extension-system-and-model-context-protocol-mcp-3ehl)
- [MCP Night 2.0: Block's Goose Layered Tool Pattern (WorkOS)](https://workos.com/blog/mcp-night-block-goose-layered-tool-pattern)

### Stripe Developer Infrastructure (Historical)
- [Stripe's Monorepo Developer Environment (Nelson Elhage)](https://blog.nelhage.com/post/stripe-dev-environment/)
- [Building and Scaling Developer Environments at Stripe (InfoQ)](https://www.infoq.com/presentations/stripe-dev-env-infrastructure/)
