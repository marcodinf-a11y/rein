# T3Code Deep Dive: Core Architecture

**March 2026**

---

## Table of Contents

1. [Repository Overview](#1-repository-overview)
2. [Monorepo Layout](#2-monorepo-layout)
3. [Tech Stack](#3-tech-stack)
4. [Server Entry Point and CLI Flags](#4-server-entry-point-and-cli-flags)
5. [Server Layer Architecture](#5-server-layer-architecture)
6. [Configuration Resolution](#6-configuration-resolution)
7. [Database and Event Sourcing](#7-database-and-event-sourcing)
8. [Web Application Architecture](#8-web-application-architecture)
9. [Desktop Application](#9-desktop-application)
10. [Contracts Package and Type Safety](#10-contracts-package-and-type-safety)
11. [Comparison with Rein Architecture](#11-comparison-with-rein-architecture)

---

## 1. Repository Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | [github.com/pingdotgg/t3code](https://github.com/pingdotgg/t3code) |
| **Stars** | ~2,500 |
| **Forks** | 292 |
| **Commits** | 889 |
| **Latest version** | v0.0.4 |
| **Primary language** | TypeScript (96.7%) |
| **License** | See repository |

T3Code is an agentic coding tool from the Ping team (known for the T3 stack). It is a monorepo application consisting of a server backend, a web-based UI, and an Electron desktop wrapper. The project leans heavily on the Effect library for typed, composable service layers on the server side.

---

## 2. Monorepo Layout

The repository uses a Turborepo-managed monorepo with two top-level directories: `apps/` for deployable applications and `packages/` for shared libraries.

```
t3code/
├── apps/
│   ├── web/            # React 19 frontend (Vite, Zustand)
│   ├── server/         # Node.js backend (Effect library)
│   ├── desktop/        # Electron wrapper
│   └── marketing/      # Marketing site
├── packages/
│   ├── contracts/      # Effect Schema types, zero runtime logic
│   └── shared/         # Git utilities and shared helpers
├── turbo.json          # Turborepo pipeline configuration
├── package.json        # Workspace root
└── bun.lockb           # Bun lockfile
```

### apps/web

The primary user interface. Built with React 19 and Vite as the bundler. State management is handled by Zustand. This app communicates with the server via API calls and likely WebSocket or SSE connections for real-time agent feedback.

### apps/server

The backend application. This is the core of T3Code's runtime — it manages agent sessions, tool execution, LLM provider communication, and persistent state. Built on Node.js with the Effect library providing the service composition layer. Entry point is `apps/server/src/main.ts`.

### apps/desktop

An Electron application that wraps the web UI for native desktop distribution. This provides system-level access (filesystem, process spawning) that a browser-based UI cannot offer directly.

### apps/marketing

A standalone marketing/landing page application. Separate from the product codebase to allow independent deployment and iteration.

### packages/contracts

A shared type definition package using Effect Schema. This package contains **zero runtime logic** — it exists purely to define the data shapes and validation schemas shared between the server and web applications. By centralizing type definitions here, the monorepo ensures that API contracts between frontend and backend remain synchronized at the type level.

### packages/shared

Shared utilities, primarily git-related helpers. These are consumed by both the server (for workspace management, diff operations) and potentially the web app (for display formatting).

---

## 3. Tech Stack

### Core Technologies

| Technology | Version | Role |
|------------|---------|------|
| **TypeScript** | 5.7.3 | Primary language across all packages |
| **Bun** | 1.3.9+ | Package manager and runtime |
| **Turborepo** | (workspace) | Monorepo build orchestration and caching |
| **Effect** | (server) | Typed functional effect system for service composition |
| **Vitest** | (testing) | Test runner across the monorepo |

### Frontend Stack (apps/web)

| Technology | Role |
|------------|------|
| **React 19** | UI framework |
| **Vite** | Build tool and dev server |
| **Zustand** | Client-side state management |

### Backend Stack (apps/server)

| Technology | Role |
|------------|------|
| **Node.js** | Server runtime |
| **Effect** | Service layer composition, error handling, dependency injection |
| **SQLite** | Persistent storage (native bindings) |

### Desktop Stack (apps/desktop)

| Technology | Role |
|------------|------|
| **Electron** | Native desktop shell |

### Build and Development

Bun serves as both the package manager (replacing npm/yarn/pnpm) and the script runner. Turborepo handles task orchestration across the monorepo — build, test, lint, and typecheck pipelines are defined in `turbo.json` with dependency-aware caching. This means `packages/contracts` is always built before `apps/server` or `apps/web` consume its types.

Vitest is the test framework throughout. Its compatibility with the Vite ecosystem means the web app tests and server tests share the same runner and configuration patterns.

---

## 4. Server Entry Point and CLI Flags

The server's entry point is `apps/server/src/main.ts`. This file is responsible for:

1. **Parsing CLI flags** — Command-line arguments configure the server's runtime behavior. These flags set values such as the listening port, workspace directory, provider configuration, and logging verbosity.

2. **Composing the Effect layer stack** — The parsed flags feed into the construction of Effect layers (see Section 5). Each flag maps to a configuration value that influences which services are instantiated and how they are wired together.

3. **Launching the runtime** — Once layers are composed, the Effect runtime is started. This boots the HTTP server, initializes the database connection, and begins accepting requests.

The CLI flag approach means the server can be started with different configurations without modifying files on disk — useful for development, testing, and CI environments. For production or persistent configuration, the flag values can be sourced from configuration files (see Section 6).

---

## 5. Server Layer Architecture

The Effect library provides T3Code's server with a layered dependency injection system. Rather than using traditional class-based DI containers, Effect layers are composable, typed values that describe how to construct services and their dependencies.

### Provider Layer

The provider layer handles LLM provider integration. This layer is responsible for:

- **Provider selection** — Choosing which LLM backend to communicate with (e.g., Anthropic, OpenAI)
- **API key management** — Sourcing and injecting credentials
- **Request formatting** — Translating T3Code's internal message format into provider-specific API payloads
- **Response streaming** — Handling streaming responses from LLM APIs and converting them into T3Code's internal event stream

The provider layer is constructed based on configuration values. Switching providers does not require changes to the layers above it — the agent logic consumes a uniform interface regardless of which LLM is behind it.

### Runtime Services Layer

The runtime services layer sits above the provider layer and contains the application's core business logic:

- **Session management** — Creating, resuming, and persisting agent sessions
- **Tool execution** — Dispatching tool calls to appropriate handlers (filesystem, shell, git operations)
- **Event sourcing** — Recording all state transitions as immutable events in SQLite
- **Workspace management** — Managing the working directory, file access, and git operations for each session

### Layer Composition

Effect layers compose bottom-up. The provider layer is constructed first, then the runtime services layer is built on top of it. The final composed layer is provided to the Effect runtime, which resolves all dependencies and starts the application.

This architecture means:

- **Testability** — Layers can be replaced with test doubles. A test can swap the real LLM provider layer with a mock that returns canned responses, while keeping all other layers intact.
- **Type safety** — If a layer requires a dependency that is not provided, the TypeScript compiler catches the error at build time. Missing services are compile-time failures, not runtime surprises.
- **Isolation** — Each layer declares its own requirements and provisions. There are no global singletons or hidden state.

---

## 6. Configuration Resolution

T3Code follows a layered configuration resolution strategy with clear precedence:

```
CLI flags (highest priority)
    |
    v
Project-level configuration
    |
    v
Global configuration
    |
    v
Built-in defaults (lowest priority)
```

### CLI Flags

Command-line arguments passed when starting the server take the highest precedence. These override all other configuration sources. Typical flags include port, workspace path, and provider selection.

### Project Configuration

Project-level configuration lives within the repository being worked on. This allows per-project customization of behavior — for example, specifying which tools are available, setting model preferences, or defining project-specific system prompts.

### Global Configuration

User-wide settings that apply across all projects unless overridden. This is where default provider credentials, preferred models, and general UI preferences are stored.

### Built-in Defaults

Hardcoded fallback values compiled into the application. These ensure the server can start with minimal configuration — sensible defaults for port, model selection, and feature flags.

### Comparison with Rein

This resolution order matches Rein's planned configuration model exactly: CLI flags > project `.rein/config.yaml` > global `~/.config/rein/config.yaml` > defaults. The convergence is unsurprising — this is the standard precedence hierarchy used by most well-designed CLI tools (Git, Docker, kubectl all follow this pattern).

---

## 7. Database and Event Sourcing

T3Code uses SQLite as its persistence layer, accessed through Node.js native bindings (not an ORM or query builder abstraction). The database schema has evolved through **13 migrations**, indicating active iteration on the data model.

### Event-Sourced Design

The database follows an event-sourced architecture. Rather than storing only the current state of a session, T3Code records the sequence of events that produced that state. This means:

- **Full history** — Every tool call, LLM response, user message, and state transition is recorded as an immutable event.
- **Replay capability** — A session's state can be reconstructed by replaying its event stream from the beginning.
- **Audit trail** — The event log provides a complete record of what happened, when, and in what order.
- **Debugging** — When something goes wrong, the event stream shows the exact sequence of operations that led to the failure.

### Why SQLite

SQLite with native bindings is a pragmatic choice for a desktop-first application:

- No separate database process to manage
- Single file per database, trivially portable
- Native bindings avoid the performance overhead of WASM-based alternatives
- WAL mode (likely enabled) supports concurrent reads with single-writer semantics

The 13 migrations suggest the schema has undergone significant evolution across the project's 889 commits. This is typical for an actively developed application that is refining its data model as requirements solidify.

---

## 8. Web Application Architecture

The web frontend (`apps/web`) is built with React 19 and uses modern patterns:

### State Management (Zustand)

Zustand is a minimal state management library. Unlike Redux, it does not require action types, reducers, or middleware boilerplate. Stores are plain JavaScript objects with methods. This choice aligns with T3Code's overall lean-tooling philosophy.

Zustand stores likely manage:

- **Session state** — Current conversation, message history, agent status
- **UI state** — Panel layouts, active views, editor state
- **Connection state** — Server connectivity, streaming status

### Build Tooling (Vite)

Vite provides fast HMR (Hot Module Replacement) during development and optimized production builds. Its native ESM support means the dev server starts instantly regardless of application size.

### React 19

The use of React 19 (the latest major version) suggests the application takes advantage of features like Server Components readiness, improved Suspense boundaries, and the `use()` hook for promise resolution in components.

---

## 9. Desktop Application

The Electron app (`apps/desktop`) wraps the web UI to provide:

- **Native filesystem access** — Direct file read/write without browser sandbox limitations
- **Process spawning** — Ability to run shell commands and manage child processes
- **System tray integration** — Background operation and quick access
- **Auto-updates** — Desktop applications can self-update without requiring users to revisit a website

The separation of `apps/web` and `apps/desktop` means the web UI can also be served standalone (e.g., connected to a remote server instance), while the desktop app bundles everything for local use.

---

## 10. Contracts Package and Type Safety

The `packages/contracts` package deserves special attention because it embodies a design principle worth examining.

### Effect Schema

Effect Schema is the type definition and validation layer from the Effect ecosystem. Unlike Zod or io-ts, Effect Schema integrates directly with Effect's runtime, providing:

- **Bidirectional codecs** — Schemas can both validate incoming data and encode outgoing data
- **Composability** — Schemas compose with Effect's pipe operator
- **Error reporting** — Validation failures produce structured, typed error values

### Zero Runtime Logic

The contracts package contains only type definitions and schemas. No business logic, no side effects, no database queries. This constraint ensures that:

1. **The package is safe to import anywhere** — No risk of pulling in server-side dependencies into the client bundle
2. **API changes are visible in version control** — Since contracts are the boundary between apps, changes to this package clearly indicate API evolution
3. **Build order is simple** — Contracts can be built first with no dependencies on other workspace packages

### Shared Type Examples

Typical contracts would include:

- Session creation/update request and response shapes
- Tool call request and result schemas
- Agent event stream message types
- Configuration value schemas

This pattern is analogous to Protocol Buffers or OpenAPI schemas, but expressed in TypeScript with runtime validation capabilities.

---

## 11. Comparison with Rein Architecture

| Aspect | T3Code | Rein |
|--------|--------|------|
| **Language** | TypeScript (96.7%) | Python |
| **Architecture** | Monorepo (Turborepo) | Single package |
| **Runtime** | Bun + Node.js | Python (uv) |
| **Service composition** | Effect layers (typed DI) | Planned: stdlib + direct composition |
| **Database** | SQLite (event-sourced, 13 migrations) | Not yet implemented |
| **UI** | React 19 + Electron desktop | CLI only (Rich) |
| **Config resolution** | CLI > project > global > defaults | CLI > project > global > defaults (identical) |
| **Type contracts** | Effect Schema in shared package | Not applicable (Python typing) |
| **Context monitoring** | Unknown (not investigated yet) | Real-time token tracking, zone-based intervention |
| **Testing** | Vitest | pytest + fixture replay |
| **Provider model** | Multi-provider (via Effect layers) | Claude Code adapter only (MVP) |

### Key Architectural Differences

**Monorepo vs. single package.** T3Code's Turborepo structure separates concerns across six packages. Rein is a single Python package. The monorepo approach makes sense for T3Code because it ships three distinct artifacts (web, desktop, server). Rein ships one (a CLI tool), so the overhead of a monorepo is not justified.

**Effect vs. direct composition.** T3Code's use of Effect for service layering provides compile-time guarantees about dependency wiring. This is a significant architectural investment — Effect has a steep learning curve and pervades the entire server codebase. Rein's simpler stdlib approach trades some type safety for lower complexity and faster onboarding.

**Event sourcing vs. planned state.** T3Code's event-sourced SQLite database is a mature persistence strategy that enables session replay and debugging. Rein has not yet implemented persistence. When it does, the event-sourced approach is worth considering, particularly for post-mortem analysis of failed tasks.

**UI investment.** T3Code ships a full React UI and Electron desktop app alongside the server. Rein is CLI-only by design. This reflects different product philosophies: T3Code targets developers who want a visual interface, while Rein targets operators who want scriptable, headless orchestration.

### Configuration Convergence

Both projects independently arrived at the same four-tier configuration resolution order. This is not a coincidence — it is the established pattern in developer tooling. The convergence validates Rein's planned configuration design.

---

## Sources

### Primary Sources
- [pingdotgg/t3code repository](https://github.com/pingdotgg/t3code) — source code and commit history
- Repository statistics: ~2.5k stars, 292 forks, 889 commits, v0.0.4

### Technology References
- [Effect documentation](https://effect.website/) — Effect library and Effect Schema
- [Turborepo documentation](https://turbo.build/repo) — monorepo build system
- [Bun documentation](https://bun.sh/) — JavaScript runtime and package manager
- [Vitest documentation](https://vitest.dev/) — test framework
