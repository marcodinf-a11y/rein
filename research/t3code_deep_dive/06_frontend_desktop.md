# T3Code Deep Dive: Frontend & Desktop

**March 2026**

This document analyzes T3Code's web frontend and Electron desktop application. The web UI is T3Code's primary differentiator against CLI-only tools like Rein, Claude Code, and Codex. Where most coding agents treat the terminal as the sole interface, T3Code invests heavily in a visual session management layer with real-time state synchronization over WebSocket.

---

## 1. Web Application Stack

The frontend is a single-page application built on a modern React stack:

| Layer | Technology | Version |
|---|---|---|
| UI framework | React | 19 |
| Routing | TanStack Router | latest |
| State management | Zustand | v8 |
| Rich text editing | Lexical | latest |
| Styling | TailwindCSS | 4 |
| Transport | WebSocket (native) | -- |

React 19 brings the transitions API and concurrent features that T3Code uses for non-blocking UI updates during long-running agent sessions. TanStack Router provides type-safe routing with search parameter validation, which matters for deep-linking into specific sessions and worktrees.

Zustand v8 with localStorage persistence means client-side state survives browser refreshes. This is a deliberate tradeoff: the server is the source of truth for session data, but UI preferences (layout, panel sizes, selected session) persist locally without round-tripping. The store is structured as a single flat object with selector-based subscriptions, avoiding the re-render storms that Redux or Context-heavy approaches suffer from in high-frequency update scenarios like streaming agent output.

Lexical replaces simpler textarea inputs for the message composer. This gives T3Code structured content editing -- users can embed code blocks, references to files, and formatted instructions directly in their prompts. Lexical's plugin architecture keeps the editor extensible without bloating the core bundle.

---

## 2. ChatView.tsx -- The 204KB Monolith

The central UI component is `ChatView.tsx` at 204KB. This single file owns session lifecycle, message display, and worktree operations. By any standard, 204KB for a React component is extreme. For context, that is roughly 5,000-6,000 lines of TypeScript -- larger than many entire applications.

### What ChatView Owns

- **Session lifecycle:** Creating sessions, resuming sessions, switching between sessions, tearing down sessions
- **Message display:** Rendering the message stream with syntax-highlighted code blocks, tool call results, plan steps, and diff views
- **Worktree operations:** Displaying worktree state, triggering merges, showing file trees for active worktrees
- **Streaming state:** Managing the incremental rendering of in-progress agent responses
- **Input handling:** The Lexical editor integration, prompt submission, interrupt buttons

### Why It Matters

A 204KB component is a maintenance liability. It suggests that T3Code's frontend grew organically around the chat view without extracting sub-components or establishing clear module boundaries. The session management, message rendering, and worktree operations are three distinct concerns that could be separated. This is likely the result of rapid iteration -- the chat view is where all the action happens, so it accumulated logic faster than refactoring could keep up.

For Rein, this is a cautionary data point. If Rein ever adds a web frontend, the session view will face the same gravitational pull. Defining module boundaries early (session state, message rendering, workspace operations) would prevent a similar monolith.

---

## 3. State Derivation Functions

T3Code uses a set of pure derivation functions to transform raw session state into UI-consumable form. These sit between the WebSocket event stream and the React render tree.

### derivePhase()

Maps the raw session state object to a UI phase enum. The phases govern what the user sees:

- **idle** -- no active session, show session picker
- **planning** -- agent is generating a plan, show plan preview
- **executing** -- agent is running tools, show live output
- **reviewing** -- execution complete, show diff review
- **error** -- something failed, show error state with retry options

This function centralizes the "what should the UI show right now" decision. Without it, every component would need to independently interpret raw session fields, leading to inconsistent UI states.

### deriveActivePlanState()

Tracks the execution progress of the agent's plan. It maps the stream of tool calls and completions back to the plan steps, computing:

- Which step is currently executing
- Which steps are complete, skipped, or failed
- Estimated progress percentage

This powers the plan progress bar and step-by-step execution view, giving users a structured understanding of what the agent is doing rather than just a raw output stream.

### extractChangedFiles()

Parses git diff information from the event stream and produces a structured list of changed files with their diff hunks. This feeds the diff review panel that appears after execution completes.

The function handles edge cases: binary files, renamed files, files changed multiple times during a session, and large diffs that need truncation for rendering performance.

### Design Pattern

All three functions are pure -- they take session state as input and return derived data. No side effects, no subscriptions. The Zustand store calls them when state changes and caches the results. This keeps the derivation logic testable and the render path predictable.

---

## 4. WebSocket Protocol

T3Code uses a single WebSocket connection per client for all communication with the backend. The protocol supports both request-response and server-push patterns.

### RPC Methods (18 total)

The request-response methods cover the full session lifecycle:

| Category | Methods |
|---|---|
| Project management | `createProject`, `listProjects`, `deleteProject` |
| Session/thread | `createThread`, `listThreads`, `getThread` |
| Messaging | `sendMessage`, `interrupt` |
| History | `rollback` |
| Configuration | `getModels`, `setModel`, `getConfig`, `updateConfig` |
| Workspace | `createWorktree`, `mergeWorktree`, `deleteWorktree` |
| System | `getHealth`, `getVersion`, `authenticate` |

Each RPC call uses a JSON envelope with a `method` field, a `params` object, and a client-generated `id` for correlating responses. This is essentially JSON-RPC 2.0 without formally declaring it as such.

### Push Channels (3)

Server-initiated messages flow on three channels:

1. **events** -- The agent output stream. Tool calls, tool results, text chunks, plan updates. This is the high-frequency channel that drives the real-time chat view.
2. **session state** -- State transitions (idle, planning, executing, reviewing, error). Lower frequency, drives the phase derivation.
3. **health** -- Backend health status, connection quality, model availability. Used for the status indicator in the UI header.

### Protocol Design

Running everything over a single WebSocket connection simplifies connection management but requires multiplexing. Each message includes a `channel` field to route it to the appropriate handler. The client maintains separate callback registrations per channel.

The protocol does not use binary framing or compression. All messages are JSON text. For the message volumes T3Code handles (agent output is already text), this is a reasonable simplicity tradeoff. Binary framing would only matter if T3Code were streaming large binary artifacts, which it does not.

---

## 5. Electron Desktop Application

T3Code ships an Electron-based desktop application that wraps the web frontend with native OS integration.

### Custom Protocol: t3://

The desktop app registers a `t3://` custom protocol for deep linking. URLs like `t3://session/abc123` or `t3://project/my-app` open the desktop app directly to the specified resource. This enables:

- Clicking links in documentation or chat that open T3Code to a specific session
- Terminal commands that launch the desktop app at a specific context
- Cross-application integration where other tools can link into T3Code

Protocol registration happens at install time via Electron's `app.setAsDefaultProtocolClient()`. On macOS, this also registers the protocol in the Launch Services database. On Linux, it writes a `.desktop` file with the protocol handler.

### Auto-Updates

The desktop app uses `electron-updater` for automatic updates. The update flow:

1. On launch, check the update server for new versions
2. Download the update in the background
3. Notify the user that an update is available
4. Apply the update on next restart

This is standard Electron auto-update machinery. The update server serves differential updates (deltas) where possible, reducing download sizes for patch releases.

### Logging

The desktop app uses rotating file logs. Log files are written to the platform-specific application data directory:

- **macOS:** `~/Library/Logs/T3Code/`
- **Linux:** `~/.config/T3Code/logs/`
- **Windows:** `%APPDATA%/T3Code/logs/`

Rotation happens by file size with a configurable retention count. This is essential for debugging desktop-specific issues (protocol handler failures, update errors, IPC problems) that would not appear in the web version.

### IPC Bridge: DesktopBridge

The `DesktopBridge` class mediates between Electron's main process and the renderer (web UI). It exposes 10 methods:

| Method | Purpose |
|---|---|
| `openFile` | Open a file in the system default editor |
| `openFolder` | Open a folder in the system file manager |
| `openTerminal` | Launch a terminal at a specific directory |
| `openExternal` | Open a URL in the default browser |
| `showNotification` | Display a native OS notification |
| `getAppVersion` | Return the current application version |
| `checkForUpdates` | Trigger a manual update check |
| `getSystemInfo` | Return OS, architecture, memory info |
| `setWindowTitle` | Update the window title bar |
| `toggleDevTools` | Open or close Chrome DevTools |

These methods are exposed to the renderer via Electron's `contextBridge.exposeInMainWorld()`, which creates a secure channel without giving the renderer full Node.js access. The web UI detects whether it is running inside Electron by checking for the presence of the `window.desktopBridge` object and conditionally enables native features.

---

## 6. Frontend as Differentiator

The web UI is T3Code's strategic moat against CLI-only tools. The investment is substantial:

- **Visual session management:** Users can see all their sessions, switch between them, review history -- operations that require mental bookkeeping in a CLI
- **Plan visualization:** The plan progress view gives users structured insight into what the agent is doing, rather than watching raw output scroll by
- **Diff review:** Integrated diff viewing with accept/reject per file, rather than requiring users to manually inspect git diffs
- **Real-time streaming:** The WebSocket protocol delivers agent output with low latency, keeping the experience responsive

For Rein, the lesson is not "build a web UI" -- that would be a massive scope expansion incompatible with the current CLI-focused MVP. The lesson is that T3Code's frontend investment reveals which operations users find painful in CLI-only workflows:

1. **Session switching** -- CLI tools lose context when you close a terminal. T3Code persists sessions server-side.
2. **Progress visibility** -- Watching raw agent output is noisy. Derived phase and plan state give structure.
3. **Diff review** -- `git diff` in a terminal is functional but not ergonomic for large changes.

Rein can address these pain points at the CLI layer: structured progress output, session persistence via workspace state, and integration with external diff tools. The goal is not to replicate T3Code's UI, but to solve the same user problems within Rein's CLI model.

---

## 7. Architectural Risks

### The ChatView Monolith

At 204KB, ChatView.tsx is a refactoring debt bomb. Any team working on the frontend will experience merge conflicts, slow IDE performance, and difficulty reasoning about the component's behavior. This is a known pattern in rapidly-developed React applications, but at this scale it becomes a blocking problem for team velocity.

### Single WebSocket Fragility

Running all communication over one WebSocket means a connection drop kills everything -- RPC responses, event streaming, and health monitoring all go dark simultaneously. T3Code likely has reconnection logic, but the protocol design makes it impossible to gracefully degrade (e.g., falling back to HTTP polling for RPC while reconnecting the event stream).

### Electron Overhead

The desktop app adds a full Chromium instance to what is fundamentally a developer tool. Memory overhead is non-trivial (200-400MB baseline). For a tool that developers run alongside their editor, browser, and terminal, this is a meaningful resource cost. The `t3://` protocol and native notifications are useful, but they may not justify the Electron tax for power users who would prefer a lighter solution.

### Zustand localStorage Persistence

Persisting Zustand state to localStorage creates a subtle data consistency risk. If the server-side session state diverges from the cached client state (e.g., after a server restart or session cleanup), the client may render stale data until the next full sync. The mitigation is straightforward -- invalidate local state on connection establishment -- but it requires discipline to implement correctly for every state slice.

---

## 8. Comparison with Rein's Approach

| Dimension | T3Code | Rein |
|---|---|---|
| Primary interface | Web UI (React SPA) | CLI |
| Session state | Server-side, streamed via WebSocket | Workspace filesystem (worktree/tempdir/copy) |
| Progress visibility | Derived phase + plan UI | Context pressure zones (green/yellow/red) |
| Diff review | Integrated diff viewer | External tools (git diff, delta, difftastic) |
| Desktop app | Electron with native integration | Not applicable |
| Resource overhead | High (Chromium + WebSocket server) | Low (CLI process only) |
| Real-time monitoring | WebSocket push | Direct process observation |

Rein's CLI-only approach trades visual richness for simplicity and resource efficiency. The two tools target different user preferences: T3Code optimizes for visual learners and teams that want a polished GUI, while Rein optimizes for terminal-native developers who want lightweight, composable tooling.

---

## 9. Key Takeaways for Rein

1. **State derivation is universally useful.** The `derivePhase()` pattern -- mapping raw state to a small set of meaningful phases -- works equally well in a CLI. Rein's context pressure zones (green/yellow/red) are already a form of phase derivation.

2. **Structured progress beats raw output.** T3Code's plan visualization succeeds because it transforms noise into signal. Rein should invest in structured progress reporting for the terminal, not just log streaming.

3. **Session persistence matters.** T3Code's server-side sessions solve a real pain point. Rein's workspace model (worktrees, tempdirs) provides session persistence through filesystem state, which is simpler but achieves the same goal.

4. **Do not build a web UI.** T3Code's 204KB ChatView and Electron overhead demonstrate the maintenance cost of a visual frontend. For Rein's scope and philosophy, the CLI is the right interface. Invest in making the CLI excellent rather than adding a GUI layer.

5. **WebSocket protocol design is instructive.** If Rein ever needs inter-process communication (e.g., a monitoring daemon), T3Code's channel-based multiplexing over a single connection is a clean pattern worth borrowing.
