# Claw Code — Deep Architecture Analysis

> A technical deep-dive into the design, systems, and tradeoffs of the `claw` CLI agent.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture & System Design](#2-architecture--system-design)
   - 2.1 [High-Level Architecture](#21-high-level-architecture)
   - 2.2 [Crate-Level Responsibilities](#22-crate-level-responsibilities)
   - 2.3 [Runtime Deep Dive](#23-runtime-deep-dive)
3. [Key Systems & Design Patterns](#3-key-systems--design-patterns)
4. [Interesting and Novel Findings](#4-interesting-and-novel-findings)
5. [Risks, Gaps, and Tradeoffs](#5-risks-gaps-and-tradeoffs)
6. [Developer Experience & Extensibility](#6-developer-experience--extensibility)
7. [Testing & Reliability](#7-testing--reliability)
8. [Suggested Improvements](#8-suggested-improvements)
9. [Next Steps](#9-next-steps)

---

## 1. Introduction

**Claw Code** (`ultraworkers/claw-code`) is a production-grade Rust reimplementation of an AI coding assistant CLI, functionally analogous to Anthropic's Claude Code. It operates as an interactive REPL-based agent capable of:

- Executing tools (bash, file operations, web fetch, LSP, MCP-bridge tools)
- Managing persistent sessions across interactions
- Enforcing a layered permission and policy system before any tool execution
- Interfacing with multiple LLM providers (Anthropic native + OpenAI-compatible endpoints)
- Supporting plugins with full lifecycle management and subprocess isolation

The repository has two distinct layers with different purposes:

| Layer | Path | Purpose |
|---|---|---|
| **Canonical Rust runtime** | `rust/` | The production `claw` binary; 9 Cargo crates |
| **Python parity workspace** | `src/` + `tests/` | Mirrored command/tool surfaces; parity audit tooling |

The Python layer is **not** a runtime. It functions as a test oracle and auditing surface for the porting effort from an upstream TypeScript implementation. The Rust workspace is the sole source of truth for shipped behavior.

---

## 2. Architecture & System Design

### 2.1 High-Level Architecture

The system uses a strict **bottom-up dependency layering**:

```
┌─────────────────────────────────────────────────────┐
│               rusty-claude-cli (claw binary)         │
│   REPL, CLI dispatch, rendering, session init        │
├───────────────────┬─────────────────────────────────┤
│     commands      │            tools                 │
│  Slash commands   │  Tool specs, execution, MCP      │
├───────────────────┴─────────────────────────────────┤
│                    runtime                           │
│  Session, conversation loop, permissions, policy,   │
│  compaction, MCP plumbing, sandbox, OAuth, hooks    │
├─────────────────────────────────────────────────────┤
│                      api                            │
│  HTTP clients, SSE streaming, provider abstraction  │
├─────────────────────────────────────────────────────┤
│      plugins         │        telemetry             │
│  Plugin lifecycle,   │  Structured tracing,         │
│  manifests, hooks    │  sinks, session traces       │
└──────────────────────┴─────────────────────────────┘
         ↕
   compat-harness  (parity tooling, not in product path)
   mock-anthropic-service  (E2E testing, not shipped)
```

**Why this separation exists:**

- `api` has no knowledge of sessions, tools, or permissions. It only speaks HTTP. This makes it trivially swappable and independently testable.
- `runtime` owns all stateful, policy-governed behavior but has **no HTTP code**. It defines `ApiClient` and `ToolExecutor` as traits — the concrete implementations live one layer above in `tools`.
- `tools` is the bridge: it implements `runtime::ApiClient` via `ProviderRuntimeClient` (which owns a `tokio::Runtime` + `api::ProviderClient`) and implements `runtime::ToolExecutor` via `SubagentToolExecutor`. This inversion keeps `runtime` free of provider-specific knowledge.
- `commands` and `rusty-claude-cli` are pure orchestration consumers. They have no business logic — they compose runtime + tools + plugins into user-facing flows.

The result is that **the conversation loop in `runtime` is entirely provider-agnostic and tool-agnostic**, operating only on traits.

---

### 2.2 Crate-Level Responsibilities

#### `rusty-claude-cli` — The binary (`claw`)

**Purpose:** Entry point, CLI argument parsing, REPL loop, terminal rendering, and system-level wiring.

**Key responsibilities:**
- Custom hand-rolled argument parser producing a `CliAction` algebraic type with variants for every CLI surface (`Repl`, `Prompt`, `Login`, `Doctor`, `Agents`, `Mcp`, `Skills`, `Init`, `Status`, `Sandbox`, `Resume`, `DumpManifests`, `Help`, …)
- `LiveCli` struct: holds the composed runtime (`BuiltRuntime`), session, model, tool allowlist, permission mode. `BuiltRuntime` implements `Drop` for ordered MCP + plugin teardown — a RAII guard for expensive subprocess resources
- `run_repl`: explicit event loop over `rustyline`-based `LineEditor` with slash-command dispatch, history, and completion
- `render.rs`: single-pass `pulldown-cmark` Markdown renderer with ANSI output, `syntect`-based code block highlighting, and an incremental `MarkdownStreamState` for safe streaming boundaries
- `input.rs`: thin rustyline adapter with slash-command `Completer`, multi-line input (Ctrl+J / Shift+Enter), and interrupt semantics
- `init.rs`: idempotent `claw init` scaffolding — detects repo stack (Rust/Python/Node/frameworks), writes `.claw.json`, merges `.gitignore`, creates `CLAUDE.md` template

**Interaction with other crates:** Depends on all other product crates. Calls `runtime::ConversationRuntime` through `BuiltRuntime`; uses `commands::SlashCommand`; uses `tools::ToolRegistry`; uses `plugins::PluginRegistry`; uses `compat_harness::ExtractedManifest` for `--dump-manifests`.

---

#### `runtime` — The core engine

**Purpose:** All stateful, policy-governed, provider-agnostic session management and conversation orchestration.

**Key responsibilities:**
- `ConversationRuntime<C: ApiClient, T: ToolExecutor>`: the main loop driver
- `Session`: durable JSONL-persisted message history with fork, compaction metadata, and rotation
- `PermissionPolicy` + `PermissionPrompter`: tool authorization gate
- `policy_engine`: lane/git-level automation rules (separate from tool permissions)
- `compact_session`: heuristic context compression
- MCP plumbing: `mcp.rs` (naming), `mcp_stdio.rs` (JSON-RPC over subprocess stdio), `mcp_client.rs` (client), `mcp_lifecycle_hardened.rs`, `mcp_tool_bridge.rs` (registry bridge)
- `sandbox.rs`: Linux `unshare`-based process isolation builder
- `oauth.rs`: authentication flows
- `hooks.rs`: `PreToolUse` / `PostToolUse` hooks from plugins
- `prompt.rs`: system prompt assembly
- `summary_compression.rs`, `compact.rs`: context window management
- `task_registry`, `task_packet`, `worker_boot`, `team_cron_registry`: agent worker orchestration

**Interaction with other crates:** Depends only on `plugins` and `telemetry`. Defines the `ApiClient` and `ToolExecutor` traits that higher layers implement. The key insight: **`runtime` never imports `api` or `tools`** — it only knows about its own traits.

---

#### `api` — Provider HTTP layer

**Purpose:** All outbound HTTP communication with LLM providers.

**Key responsibilities:**
- `AnthropicClient`: Anthropic Messages API with SSE streaming, prompt cache headers, retry/backoff
- `OpenAiCompatClient`: OpenAI-compatible endpoints (xAI, local servers, etc.)
- `ProviderClient` enum: closed union over provider kinds — `Anthropic | Xai | OpenAi`
- `sse.rs`: raw SSE event parsing
- `providers/`: `detect_provider_kind`, `resolve_model_alias`, `max_tokens_for_model`
- `prompt_cache.rs`: cache tracking and header injection
- `telemetry` re-export for HTTP request/response events

**Interaction with other crates:** Depends on `runtime` (for shared types) and `telemetry`. Is imported only by `tools` and `rusty-claude-cli`. The `api::AnthropicClient` is also aliased as `api::ApiClient` — this is a naming collision with `runtime::ApiClient` (the trait) that is manageable but worth noting.

---

#### `tools` — Tool definitions and execution bridge

**Purpose:** The critical glue layer: defines all built-in tool specs, implements `runtime`'s traits using `api`'s clients, and routes tool calls to their handlers.

**Key responsibilities:**
- `ToolSpec` / `ToolRegistry` / `GlobalToolRegistry`: tool metadata (name, description, JSON schema), registration, and global singletons via `OnceLock`
- `ProviderRuntimeClient`: implements `runtime::ApiClient` — owns a Tokio runtime and `api::ProviderClient`, translates `ApiRequest` → `MessageRequest`, maps stream events back to `AssistantEvent`
- `SubagentToolExecutor`: implements `runtime::ToolExecutor` — dispatches tool names to `execute_tool_with_enforcer`
- `execute_tool_with_enforcer`: the main tool dispatch `match` — routes to bash, file ops, web fetch, task/team/cron/MCP helpers, etc.
- MCP bridge: `run_mcp_tool` uses the global `McpToolRegistry` to JSON-RPC call external MCP servers
- `OnceLock` global registries for LSP client, MCP bridge, team/cron/task/worker — shared across tool calls in a session

**Interaction with other crates:** Depends on `api`, `runtime`, `plugins`. This is intentionally the highest-complexity crate — it integrates across all layers.

---

#### `commands` — Slash command system

**Purpose:** User-facing `/command` surface — definitions, handlers, and REPL-visible help.

**Key responsibilities:**
- `SlashCommandSpec` table: all `/`-prefixed commands (`/help`, `/status`, `/sandbox`, `/compact`, `/model`, `/permissions`, `/mcp`, `/session`, `/config`, `/resume`, etc.)
- Command handlers that compose `runtime` primitives: e.g. `/compact` calls `compact_session(session, CompactionConfig { … })`; `/mcp` reads MCP config from `runtime` config loader
- Manifest generation (`SlashCommandManifest`) for `--dump-manifests`

**Interaction with other crates:** Depends on `plugins`, `runtime`. Imported by `rusty-claude-cli` only.

---

#### `plugins` — Plugin lifecycle and manifests

**Purpose:** Manifest-driven plugin system with full lifecycle, hooks, and subprocess tool isolation.

**Key responsibilities:**
- `PluginManifest`: serde struct for `.claude-plugin/plugin.json` — permissions, hooks, lifecycle commands, tool definitions
- `Plugin` trait: `metadata`, `hooks`, `lifecycle`, `tools`, `validate`, `initialize`, `shutdown`
- Three implementations: `BuiltinPlugin`, `BundledPlugin`, `ExternalPlugin` — differ on path validation and install semantics
- `PluginRegistry`: sorted entries, aggregated hooks and tools with duplicate detection, bulk initialize/shutdown
- `PluginManager`: install from `LocalPath` or `GitUrl`, persist `installed.json`, resolve `externalDirectories`
- Plugin tools execute as **subprocesses** with `CLAWD_*` env vars and JSON on stdin — the security boundary is OS-level process isolation

**Interaction with other crates:** No product crate dependencies (only `serde`/`serde_json`). Imported by `runtime`, `tools`, `commands`, `rusty-claude-cli`.

---

#### `telemetry` — Structured observability

**Purpose:** Lightweight, structured event tracing across HTTP calls and sessions.

**Key responsibilities:**
- `ClientIdentity` / `AnthropicRequestProfile`: request header building and `extra_body` JSON merging
- `TelemetryEvent`: tagged JSON union for HTTP lifecycle, analytics, session traces
- `TelemetrySink` trait: `MemoryTelemetrySink` (tests) + `JsonlTelemetrySink` (production append)
- `SessionTracer`: per-session sequence counter; each event gets a merged trace record

**Interaction with other crates:** No product dependencies. Imported by `api` and `runtime`.

---

#### `compat-harness` — TypeScript upstream extraction

**Purpose:** Parity tooling only — not in the production binary's critical path.

**Key responsibilities:**
- `UpstreamPaths::resolve_upstream_repo_root`: walks candidate directories (`CLAUDE_CODE_UPSTREAM`, sibling `claw-code`, `reference-source/claw-code`, `vendor/claw-code`) to find a tree where `src/commands.ts` exists
- `extract_manifest`: reads `commands.ts`, `tools.ts`, `cli.tsx` as plain strings; cheap line-oriented heuristics to extract `CommandRegistry`, `ToolRegistry`, `BootstrapPlan` without a TS parser
- Used by `claw --dump-manifests` for parity comparison and documentation

**Interaction with other crates:** Depends on `commands`, `tools`, `runtime` (for shared types). Imported by `rusty-claude-cli`.

---

#### `mock-anthropic-service` — Deterministic E2E mock

**Purpose:** A local HTTP server that impersonates the Anthropic Messages API with scripted, scenario-driven responses for parity testing.

**Key responsibilities:**
- Binds TCP, accepts connections, parses HTTP `POST /v1/messages` bodies as `MessageRequest`
- `detect_scenario`: scans the latest user message for `PARITY_SCENARIO:<name>` token to select a `Scenario` variant (12 scenarios across baseline, file-tools, permissions, multi-tool-turns, bash, plugin-paths, session-compaction, token-usage categories)
- Returns scripted SSE streams or non-streamed JSON responses per scenario
- Records `CapturedRequest` entries; exposes them for post-run assertion

**Interaction with other crates:** Depends on `api`. Used only in dev/test (`rusty-claude-cli`'s dev dependencies and integration test binary).

---

### 2.3 Runtime Deep Dive

#### Session Lifecycle

`Session` in `runtime/session.rs` is the durable state container. Key design decisions:

- **JSONL persistence** (not JSON snapshots): the session file is append-only. Each `push_message` call appends one JSONL line after the initial meta + compaction header is written. Full snapshots are only written on explicit save. This means the on-disk format is a replay log, which is crash-resilient.
- **File rotation** at 256 KiB with 3 `.rot-*.jsonl` rotated files — prevents unbounded growth in long sessions.
- **Fork semantics**: `session.fork()` creates a new `session_id` with copied messages and a `SessionFork { parent_session_id, branch_name }` provenance link — enabling resumable branching.
- **Load path duality**: `load_from_path` detects whether the file is a single JSON object with `messages` (legacy snapshot format) or JSONL lines (current format) and handles both.

#### Conversation Loop + Compaction Strategy

`ConversationRuntime::run_turn` in `runtime/conversation.rs` is the core agent loop:

```
run_turn(user_input):
  push user message to session
  loop (up to max_iterations):
    build ApiRequest from session.messages
    stream → AssistantEvent[]
    fold events into ConversationMessage (require MessageStop + non-empty content)
    update UsageTracker
    append assistant message to session
    if no ToolUse blocks → break
    for each ToolUse:
      PreToolUse hook → PermissionPolicy::authorize_with_context
      if denied → synthesize tool_result with denial message
      if allowed → ToolExecutor::execute + PostToolUse hook
  maybe_auto_compact()
  return TurnSummary
```

**Compaction** (`runtime/compact.rs`) is entirely heuristic — there is no LLM-based summarization. The strategy:

1. Keep the last `preserve_recent_messages` (default: 4) messages intact.
2. Condense the removable prefix into a single synthetic `system`-role message containing: a `<summary>` block with statistics (message count, tool calls, file paths, user intent history, "pending work" heuristics), a timeline of removed messages, and the prior compacted summary if this is a second compaction.
3. `maybe_auto_compact` (triggered when `cumulative_input_tokens ≥ threshold`, default 100,000) uses `max_estimated_tokens: 0` — this causes `should_compact` to skip the token-count gate and compact whenever there are more messages than `preserve_recent_messages`. Effectively: **after 100K tokens, every turn compacts aggressively**.

The `/compact` slash command exposes this manually with configurable thresholds.

#### Permission Enforcement + Policy Engine

There are **two orthogonal systems** named similarly:

**`runtime/permissions.rs`** — per-tool authorization in the conversation loop:
- `PermissionMode` is an ordered enum: `ReadOnly < WorkspaceWrite < DangerFullAccess`, plus `Prompt` and `Allow` as special modes.
- `PermissionPolicy` holds per-tool required modes and string-pattern allow/deny/ask rules with a rich syntax: `bash(git:*)` — tool name before `(`, pattern inside (prefix glob, exact, or `*`).
- Evaluation order: deny rules win first → hook overrides (Allow/Deny/Ask from `PreToolUse` plugin hooks) → ask rules → allow rules or mode-based authorization. No prompter present when prompt required → deny with reason.

**`runtime/policy_engine.rs`** — lane/git automation governance:
- Operates on `LaneContext` (green level, branch staleness, blockers, review status, diff scope) rather than individual tool calls.
- `PolicyRule { condition, action, priority }` with combinatorial boolean conditions and `PolicyAction` variants like `MergeToDev`, `RecoverOnce`, `Escalate`, `CloseoutLane`, `Reconcile`.
- This is for multi-agent worker orchestration, not user-facing permission prompts.

#### Sandbox Model

`runtime/sandbox.rs` builds Linux `unshare`-based isolation commands:

- Namespace isolation: `--user --map-root-user --mount --ipc --pid --uts --fork` (optionally `--net`)
- `HOME` and `TMPDIR` are redirected under the workspace for filesystem containment
- `SandboxStatus` probes `unshare --user` availability on first use (cached in `OnceLock`) and detects container environments via `/.dockerenv`, `/run/.containerenv`, and `/proc/1/cgroup`
- Filesystem isolation modes (`off`, `workspace-only`, `allow-list`) are communicated to the inner shell via environment variables — enforcement is delegated to the child process, not enforced by kernel namespaces alone

#### MCP Integration

MCP (Model Context Protocol) is integrated at multiple depths:

- **`runtime/mcp.rs`**: naming utilities — `mcp_tool_name(server, tool)` produces `mcp__<server>__<tool>` qualified names; `unwrap_ccr_proxy_url` strips Claude proxy wrappers; `scoped_mcp_config_hash` fingerprints configs for deduplication.
- **`runtime/mcp_stdio.rs`**: JSON-RPC over subprocess stdio — spawns MCP server processes, manages the communication channel.
- **`runtime/mcp_client.rs`**: client-side MCP protocol logic.
- **`runtime/mcp_lifecycle_hardened.rs`**: resilient lifecycle management — start/stop/restart of MCP server processes.
- **`runtime/mcp_tool_bridge.rs`** + **`tools::run_mcp_tool`**: the runtime bridge that routes `mcp__*` tool names through `McpToolRegistry::call_tool` → `McpServerManager` JSON-RPC `tools/call`.
- **`LiveCli`'s `RuntimeMcpState`** in `rusty-claude-cli`: holds the embedded Tokio runtime and `McpServerManager` shared across the session; cleaned up via `Drop` on `BuiltRuntime`.

The practical effect: from the conversation loop's perspective, MCP tools look identical to built-in tools. They are registered in `GlobalToolRegistry`, appear in `ToolSpec` lists sent to the model, and are dispatched through the same `ToolExecutor::execute` path.

#### Tool Execution Pipeline

```
ConversationRuntime receives ToolUse { id, name, input }
  → PreToolUse hooks (plugins)
  → PermissionPolicy::authorize_with_context(tool_name, input)
      → deny/ask/allow rules + PermissionPrompter (if Prompt mode)
  → ToolExecutor::execute(tool_name, input_json)
      [SubagentToolExecutor in tools crate]
      → execute_tool_with_enforcer(name, input)
          → bash / file_read / file_write / web_fetch / mcp / plugin_tool / ...
          → maybe_enforce_permission_check (for workspace-touching tools)
  → PostToolUse hook (plugins)
  → result appended as ToolResult content block
```

---

## 3. Key Systems & Design Patterns

### Plugin System

The plugin system is **manifest-driven and subprocess-isolated**:

- Discovery: `.claude-plugin/plugin.json` at the plugin root (constant `MANIFEST_RELATIVE_PATH`)
- `PluginManifest` is a rich serde struct: tools (with JSON schema input), hooks (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`), lifecycle (`Init`, `Shutdown`), required permissions
- Plugin tools are **not** in-process function calls — they execute as child processes with:
  - `CLAWD_TOOL_NAME`, `CLAWD_SESSION_ID`, `CLAWD_WORKSPACE_DIR`, `CLAWD_PERMISSION_MODE` env vars
  - JSON input on stdin
  - Plugin output read from stdout
- This means a misbehaving plugin cannot corrupt the agent runtime's memory — the process boundary is the security boundary
- `PluginRegistry::aggregated_hooks` and `aggregated_tools` collect across all enabled plugins, with duplicate tool name detection surfaced as `PluginManifestValidationError`

### Tool Abstraction Layer

Tools are defined as `ToolSpec { name, description, input_schema: serde_json::Value }` — the schema is a raw JSON Schema object passed directly to the model. This means:

- Adding a tool requires only defining a `ToolSpec` and a match arm in `execute_tool_with_enforcer`
- The model's understanding of the tool comes entirely from the description and schema
- `GlobalToolRegistry` uses `OnceLock<Arc<Mutex<GlobalToolExecutor>>>` for a lazily-initialized singleton that is reused across tool calls

### Provider Abstraction

`ProviderClient` is a **closed enum** rather than a trait object:

```rust
pub enum ProviderClient {
    Anthropic(AnthropicClient),
    Xai(AnthropicClient),      // xAI uses Anthropic-compatible API
    OpenAi(OpenAiCompatClient),
}
```

The adapter in `tools::ProviderRuntimeClient` bridges this to `runtime::ApiClient`. Model alias resolution happens in `providers::resolve_model_alias` — `sonnet`, `opus`, `haiku` are mapped to versioned model IDs per provider.

A providers module also has an optional `Provider` trait for async streaming, but it is not currently wired into the main dispatch path (the enum dispatch is used instead).

### Streaming + SSE Handling

- `api/src/sse.rs` handles raw Server-Sent Events parsing
- `AssistantEvent` is the normalized representation: `TextDelta`, `ToolUse { id, name, input }`, `Usage`, `PromptCache`, `MessageStop`
- `MarkdownStreamState` in the CLI's `render.rs` uses `find_stream_safe_boundary` to only flush output after line boundaries that are safe in Markdown context (not mid-fence, not mid-emphasis) — avoiding visual artifacts in streaming display
- The conversation loop collects the full event stream before processing, not a true incremental push. Streaming is used at the HTTP level for latency, not for partial tool dispatch.

### REPL Interaction Model

The REPL is an explicit event loop rather than a framework-driven abstraction:

```
loop:
  editor.set_completions(cli.repl_completion_candidates())
  match editor.read_line():
    Submit(input):
      if slash command → handle_repl_command
      else → cli.run_turn(input)
    Cancel → continue
    Exit → persist_session + break
```

`SlashCommandHelper` implements the full rustyline `Helper` suite to provide tab completion only for `/`-prefixed input when the cursor is at end-of-line — preventing completion noise during normal prose input.

### Compatibility / Parity System

Three-layer parity infrastructure:

1. **`compat-harness`** (Rust): static string analysis of the upstream TypeScript source to extract command/tool/bootstrap manifests. Used by `claw --dump-manifests` to compare what exists in the Rust implementation against what existed in the TS original.

2. **Python parity workspace** (`src/`): mirrors the command/tool surface in Python with JSON snapshots under `src/reference_data/`. `parity_audit.py` compares current Python module coverage against archived TS file mappings. Tested by `tests/test_porting_workspace.py` which asserts `PORTED_COMMANDS ≥ 150`, `PORTED_TOOLS ≥ 100`, and ≥ 28 directory mappings.

3. **`mock-anthropic-service`** + `mock_parity_harness.rs`: 12 behavioral scenarios tested end-to-end against the real `claw` binary in a clean subprocess environment. Assertions cover JSON output shape, iteration counts, tool use/result counts, auto-compaction events, token usage, and estimated cost fields.

---

## 4. Interesting and Novel Findings

### 1. Trait-Based Provider/Tool Inversion

The `runtime` crate defines `ApiClient` and `ToolExecutor` as traits but never implements them — the implementations live in `tools`. This is a deliberate inversion: the core loop is tested with `StaticToolExecutor` (a `BTreeMap` of closures) and mock clients without ever touching HTTP or OS processes. The design makes unit-testing the conversation logic trivial and completely decoupled from provider details.

**Why it matters:** Most agent frameworks embed the HTTP client in the conversation loop. Here, swapping providers or adding a mock requires only implementing two traits, not forking the loop.

### 2. Scenario-Token Protocol in Mock Service

The mock Anthropic service uses an in-band protocol: the client sends `PARITY_SCENARIO:<name>` as a token in the user message, and the server uses `detect_scenario` to select a scripted response. This means:

- No out-of-band configuration file needed per test
- The test can drive scenario selection through the exact same code path as production prompts
- Multi-turn scenarios work naturally because subsequent requests will not contain the token, so the server responds with the scenario's "second turn" behavior

**Why it matters:** It eliminates the test harness from the call path — the CLI binary under test is completely unaware it is being tested.

### 3. Heuristic Compaction (No LLM Round-Trip)

Context compaction generates a `<summary>` block using only string operations on the existing transcript — extracting tool names used, file paths touched, user questions asked, and "pending work" markers. It does not call the model to summarize. This is:

- **Cheaper**: no extra API call and associated latency/cost
- **Deterministic**: the same session always compacts to the same summary
- **Testable**: the mock parity harness includes a `session-compaction` scenario that asserts `auto_compaction` appears in output JSON

The tradeoff is quality: the heuristic summary loses semantic coherence compared to an LLM-generated summary. But for context window management in a CLI tool, correctness of the summary is less critical than speed and cost.

### 4. Two-Domain Permission Architecture

The repository has two completely separate permission systems that share no code:

- `runtime/permissions.rs`: governs individual tool call authorization within a conversation turn (user-facing, interactive)
- `runtime/policy_engine.rs`: governs lane/branch automation decisions for multi-agent orchestration (machine-facing, rule-based)

Separating these prevents the complexity of multi-agent coordination rules from leaking into the interactive user flow, and vice versa. The `LaneContext` used by `policy_engine` has no overlap with the `PermissionRequest` / `PermissionContext` used in the conversation loop.

### 5. RAII-Based Subprocess Resource Management

`BuiltRuntime` in `rusty-claude-cli` implements `Drop` to:
1. Shut down MCP server subprocesses via `RuntimeMcpState`
2. Invoke `shutdown` on all enabled plugins via `PluginRegistry`

This means the operator doesn't need to explicitly manage cleanup — Rust's ownership system guarantees it runs on any exit path, including panics and early returns. For a CLI managing multiple external subprocesses (MCP servers, plugin lifecycle commands), this is the correct pattern and prevents orphaned processes.

### 6. Session Fork with Provenance

`session.fork()` creates a new session with a `SessionFork { parent_session_id, branch_name }` link rather than a copy-and-forget. This enables audit trails of branched conversations, which is non-trivial to reconstruct after the fact if the fork is opaque.

### 7. Config Hash for MCP Deduplication

`mcp.rs::scoped_mcp_config_hash` computes a stable 64-bit hash of an MCP server configuration. This is used to deduplicate MCP server instances — if two configuration entries hash to the same value, only one server process is started. This prevents the common mistake of spawning duplicate MCP subprocesses when configs are loaded from multiple sources (user settings + project settings + local overrides).

---

## 5. Risks, Gaps, and Tradeoffs

### Architectural Risks

**`tools/src/lib.rs` is a monolith.** At thousands of lines, it is the single largest file in the codebase and owns: tool spec definitions, two trait implementations (`ApiClient`, `ToolExecutor`), all tool handler logic, global registries, MCP bridge routing, and plugin tool dispatch. This creates high coupling and makes the file difficult to navigate. A future contributor adding a tool must understand the full file to find the right insertion point.

**`ProviderClient` is a closed enum.** Adding a new provider requires modifying `api/src/providers/` and the `ProviderClient` enum. There is an unused `Provider` trait in the codebase that suggests this was considered but not completed. Until the enum is replaced with trait objects or an open registry, provider extensibility requires code changes in the `api` crate.

**Filesystem isolation is advisory, not enforced.** `sandbox.rs` builds `unshare`-based isolation commands, but the filesystem isolation modes (`workspace-only`, `allow-list`) are communicated via environment variables to the child shell, not enforced by kernel filesystem namespaces. A child process that ignores these env vars can still access the full filesystem. True enforcement would require seccomp or a bind-mount strategy.

### Complexity Risks

**MCP integration spans 6+ modules in `runtime` alone.** `mcp.rs`, `mcp_client.rs`, `mcp_stdio.rs`, `mcp_lifecycle_hardened.rs`, `mcp_tool_bridge.rs`, plus routing in `tools` and state management in `rusty-claude-cli`. This depth is appropriate for production MCP support, but a regression in the JSON-RPC layer could manifest in subtle ways (tool call returns partial data, server process leaks). The hardened lifecycle module suggests this has already been a pain point.

**`policy_engine.rs` complexity vs. current usage.** The lane/branch automation policy engine is sophisticated — boolean rule combinators, prioritized actions, `LaneContext` with 10+ boolean fields. However, the current production binary primarily uses it for worker orchestration features that are still maturing (per `ROADMAP.md`). There's a risk of maintaining a complex rule engine whose full surface isn't exercised in tests.

**Parity documentation drift.** `PARITY.md` references 9-lane checkpoints, 19 captured requests, and 10 scenarios. The actual harness implements 12 scenarios and expects 21 requests. Treating `PARITY.md` as a specification creates false confidence — it should be treated as a changelog, not a spec.

### Performance Risks

**Blocking `reqwest` in `tools`.** The `tools` crate depends on `reqwest` with the `blocking` feature in addition to async. Blocking HTTP calls in a Tokio async context can starve the executor. These should be audited to ensure they run on `spawn_blocking` threads, not inline in async tasks.

**Compaction token estimate is `chars/4`.** This is a well-known approximation but fails for non-Latin text (CJK characters are 1 char but ~1-3 tokens). Sessions with non-English content may compact later than intended, or auto-compaction may not trigger at the right time.

**`OnceLock` global registries in `tools`.** Global mutable state in `Arc<Mutex<GlobalToolExecutor>>` is correct for safety but creates a serialization point for all tool calls. In a future multi-session or multi-agent scenario, this becomes a contention bottleneck.

### Plugin / Tool Execution Risks

**Plugin tool processes inherit the agent's environment.** While `CLAWD_*` env vars are injected, there is no filtering of the parent process's environment variables before spawning plugin tools. A plugin could read `ANTHROPIC_API_KEY` from env — arguably expected for a tool that might call APIs, but it means plugins have full credential access.

**No timeout on plugin tool execution.** The subprocess-based plugin tool model has no configurable timeout in the manifest. A hung plugin tool will block the conversation turn indefinitely.

### Parity Approach Limitations

The Python parity workspace maintains `PORTED_COMMANDS ≥ 150` and `PORTED_TOOLS ≥ 100` as thresholds, but these are **count assertions, not behavioral assertions**. A command being "ported" in the Python mirror means it exists as a Python function; it doesn't mean the behavior matches the Rust implementation. The behavioral parity guarantee comes from the mock harness, which covers only 12 scenarios.

---

## 6. Developer Experience & Extensibility

### Adding a New Built-In Tool

1. Define a `ToolSpec` in `tools/src/lib.rs` (name, description, JSON schema)
2. Add it to `mvp_tool_specs()` or the appropriate registry function
3. Add a match arm in `execute_tool_with_enforcer` with the handler logic
4. If the tool needs permissions, call `maybe_enforce_permission_check` with the appropriate `PermissionMode`
5. Add a parity scenario in `mock_parity_scenarios.json` and a corresponding `Scenario` variant in `mock-anthropic-service` for E2E coverage

**Friction:** Step 3 is in a very large `match` block inside an already large file. The lack of a registration mechanism (e.g., a `#[tool]` attribute macro or a `impl Tool for MyTool` pattern) means the file grows unbounded.

### Adding a New Slash Command

1. Add a `SlashCommandSpec` entry in `commands/src/lib.rs`
2. Add a match arm in the command handler
3. Add a `SlashCommand` variant if it takes arguments

**Friction:** Low — the commands crate is well-isolated. The only risk is missing the completion registration in `rusty-claude-cli`'s `repl_completion_candidates`.

### Adding a New Provider

1. Implement a new client struct in `api/src/providers/`
2. Add a variant to `ProviderClient`
3. Add model aliases to `resolve_model_alias`
4. Implement the stream/send-message methods

**Friction:** Medium. The closed enum means touching 3+ files. If the `Provider` trait were completed and wired into `ProviderClient`, this would be easier.

### Extending MCP Capabilities

The MCP integration is deep but the extension surface is the MCP protocol itself — adding new MCP servers is done via config (`~/.claw.json` or `.claw.json`), not code. Adding support for new MCP transport types (e.g., HTTP instead of stdio) would require additions to `mcp_stdio.rs` and `mcp_lifecycle_hardened.rs`.

### Building Plugins

The plugin system has a clear contract:
- Create a directory with `.claude-plugin/plugin.json`
- Define tools with JSON schemas and executable paths
- Define hooks as executable paths that receive/return JSON
- Point `externalDirectories` or install via `claw plugins install`

**Friction:** The subprocess tool model is clean but has no SDK. Plugin authors must implement the JSON stdin/stdout contract manually. A TypeScript or Python helper library would significantly reduce friction.

---

## 7. Testing & Reliability

### Rust Integration Testing Strategy

Tests are split into two levels:

**Module-level tests** within `runtime` (particularly `conversation.rs` with `StaticToolExecutor` and mock `ApiClient` implementations) test the conversation loop, permission evaluation, and compaction logic in isolation. These are fast, deterministic, and cover the critical path.

**Binary-level integration tests** in `rusty-claude-cli/tests/` test the actual `claw` binary as a black box:

- `mock_parity_harness.rs`: 12 scenarios, clean subprocess environment with `env_clear()`, asserts JSON output shape. This is the most thorough behavioral test.
- `cli_flags_and_config_defaults.rs`: flag parsing, config merge precedence, session resume, `doctor` output — covers the CLI surface orthogonal to provider behavior.
- `output_format_contract.rs`: `--output-format json` shape validation.
- `resume_slash_commands.rs`: `/resume` + slash command interaction.

### Role of `mock-anthropic-service`

The mock service is **the regression safety net for behavioral correctness**. By scripting 12 specific scenarios — including multi-turn tool loops, permission denial flows, auto-compaction triggers, plugin paths, and token/cost fields — the harness can detect:

- Changes in iteration count or tool dispatch order
- Regressions in JSON output schema
- Auto-compaction not triggering (or triggering incorrectly)
- Permission enforcement failures

The clean-environment subprocess test (`env_clear()` + minimal PATH) means the binary is tested in a configuration close to a fresh install, not an arbitrary developer environment.

### Python Parity Tests

`tests/test_porting_workspace.py` validates the Python mirror workspace rather than the Rust binary. Its assertions:
- Surface coverage (`PORTED_COMMANDS ≥ 150`, `PORTED_TOOLS ≥ 100`)
- Subprocess output of `python -m src.main summary`, `parity-audit`, `commands`, `tools`, etc.
- Turn loop simulation, bootstrap session, execution registry

These tests are not behavioral equivalence tests — they validate the Python mirror's own consistency, not Rust/Python parity.

### CI Pipeline

`.github/workflows/rust-ci.yml` runs four jobs with path filtering:

1. **`doc-source-of-truth`**: Python doc checker script (ensures key docs are consistent)
2. **`fmt`**: `cargo fmt --all --check` — zero tolerance for formatting divergence
3. **`test-workspace`**: `cargo test --workspace` — all crates, all integration tests
4. **`clippy-workspace`**: `cargo clippy --workspace` — note: the CI file does not pass `-D warnings`, while `CLAUDE.md` instructs developers to use `-- -D warnings` locally. This means clippy warnings can pass CI but will be flagged in manual verification.

Confidence level: **High for covered scenarios, moderate for edge cases.** The mock harness covers the happy path and several failure modes well. The main gaps are: concurrent session behavior, MCP server lifecycle edge cases (restart, timeout, malformed responses), and filesystem isolation effectiveness.

---

## 8. Suggested Improvements

### High Impact

**1. Split `tools/src/lib.rs` into a module tree.**
Extract tool handler logic into per-tool modules under `tools/src/builtins/` (e.g. `bash.rs`, `file.rs`, `web.rs`, `mcp.rs`). Keep `lib.rs` as a thin registry and dispatch facade. This is the single highest-leverage structural improvement for long-term maintainability.

**2. Complete the `Provider` trait in `api`.**
Replace the `ProviderClient` closed enum with the existing (unused) `Provider` trait and dynamic dispatch or an open registry. This would make adding providers a zero-code-change operation in the core crates and reduce the coupling between `api` and `tools`.

**3. Add plugin tool timeouts.**
Add a `timeout_seconds: Option<u64>` field to `PluginToolManifest` and enforce it when spawning plugin tool subprocesses. Without this, a hung plugin is unrecoverable without killing the entire agent process.

### Medium Impact

**4. Enforce filesystem isolation in sandbox with bind mounts.**
The current `workspace-only` mode communicates the restriction via env vars, but doesn't enforce it. A proper implementation would use `--bind` mounts in the `unshare` command to mount only the workspace directory into the container's filesystem view.

**5. Migrate compaction token estimate to a tokenizer.**
Use `tiktoken-rs` or a simple byte-pair encoding approximation to produce accurate token counts for compaction decisions. The `chars/4` heuristic is fine for English but problematic for multilingual sessions.

**6. Align `PARITY.md` to the current harness state.**
Update `PARITY.md` to reflect 12 scenarios and 21 expected requests. Add a CI step that validates the count of scenarios in `mock_parity_scenarios.json` matches the `Scenario` enum in `mock-anthropic-service`. This closes the documentation drift risk.

**7. Add a plugin SDK.**
Create a minimal Python and/or TypeScript helper library that implements the `CLAWD_*` env contract and JSON stdin/stdout protocol. Even a 50-line helper reduces plugin author friction substantially.

### Low Impact

**8. Add `#[non_exhaustive]` to `PermissionMode`.**
The `PermissionMode` enum's ordered semantics (`ReadOnly < WorkspaceWrite < DangerFullAccess`) are implementation-dependent. Marking it `#[non_exhaustive]` prevents downstream crates from `match`-exhausting on it, making future additions non-breaking.

**9. Replace `env_clear()` in tests with an explicit allowlist.**
The `env_clear()` + manual env reconstruction in the parity harness works but is fragile — if the binary depends on a new env var, tests silently start failing. An explicit `AllowedEnv` list with documentation is more maintainable.

**10. Add session integrity checksums.**
Each JSONL session file line could include a `sha2` hash of the content for corruption detection on load. The `sha2` dependency is already present in `runtime`.

---

## 9. Next Steps

### For a New Contributor

1. **Start with `rust/crates/runtime/src/conversation.rs`.** Read `ConversationRuntime::run_turn` from top to bottom. This is the single most important function in the codebase — understanding it gives you the skeleton of everything else.

2. **Run the parity harness.** Follow `USAGE.md` to build the binary, then run `cargo test mock_parity` in `rust/crates/rusty-claude-cli/`. Read `tests/mock_parity_harness.rs` alongside `rust/mock_parity_scenarios.json` and the `Scenario` enum in `mock-anthropic-service/src/lib.rs`. This gives you a working mental model of the full stack.

3. **Add a trivial tool.** Adding a `current_time` tool that returns `chrono::Local::now().to_string()` requires touching only `tools/src/lib.rs`. This is the fastest path to understanding the tool registration + execution flow end-to-end.

### Highest Leverage Improvements

In priority order for production hardening:

1. **Split `tools/src/lib.rs`** — reduces onboarding time for every future contributor and makes the tool dispatch logic easier to reason about in code review.
2. **Plugin tool timeouts** — prevents a class of hard-to-debug hangs in production use.
3. **Complete `Provider` trait** — unblocks provider ecosystem growth without core crate changes.
4. **Parity doc alignment** — maintains trust in `PARITY.md` as a reliable status document.

### Directions for Evolution

**Multi-session concurrency.** The `OnceLock` global registries in `tools` and the single-session focus of `LiveCli` suggest the current architecture supports one conversation at a time per process. Adding concurrent session support would require converting globals to per-session state and potentially separating the Tokio runtime from `ProviderRuntimeClient`.

**Remote/distributed agent mode.** `runtime` already contains `remote.rs` and `worker_boot.rs`, and `PHILOSOPHY.md` positions the system as a "coordination demo" for multi-agent flows (OmX/clawhip). The `policy_engine` and `lane_events` modules are foundations for this. The natural next step is a stable inter-agent communication protocol built on top of the existing `task_packet` / `task_registry` types.

**Native MCP server authoring.** Currently MCP servers are external subprocesses. A lightweight in-process MCP server registration path would allow `claw` plugins to expose MCP tools without a separate binary, reducing operational complexity for plugin authors.

**Observability hardening.** The `telemetry` crate has `JsonlTelemetrySink` and `SessionTracer`. Wiring these to an OpenTelemetry exporter would make the agent's behavior inspectable in standard observability stacks, which becomes critical at production scale.

---

*Analysis based on source at `ultraworkers/claw-code`. Last reviewed against the current `main` branch.*
