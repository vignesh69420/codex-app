# Codex-App — Complete Architectural Knowledge Base

> Deep, exhaustive architectural breakdown of the repository `vignesh69420/codex-app`
> (a fork of OpenAI **Codex**). This document maps every major folder, crate, module,
> feature, workflow, integration, and dependency, and traces all execution flows
> end‑to‑end.

---

## 0. Executive Summary & The "Desktop App" Clarification

**What this repository actually is:** Codex is an OpenAI **coding agent that runs locally**. It is **not** an Electron/JS desktop application. It is a **Rust monorepo** (`codex-rs`, ~92 crates) that compiles to a single multi-call binary (`codex`) plus **TypeScript and Python SDKs** (`sdk/`) and an **npm launcher** (`codex-cli/`).

There are **four product surfaces**, all backed by the same engine:

| Surface | How it runs | Frontend | Backend |
|---|---|---|---|
| **CLI / TUI** (default) | `codex` | `codex-rs/tui` (ratatui terminal UI) | in‑process `app-server` → `core` |
| **Desktop App** (`codex app`) | native installed app (downloaded separately; **not in this repo**) | native (Swift/Win32) | spawns/embeds `app-server` (JSON‑RPC) |
| **IDE Extensions** (VS Code/Cursor/JetBrains) | extension process | TS using `app-server-protocol` types | `app-server` over stdio/UDS/WS |
| **SDK / Headless** | `codex exec` | TS SDK (`sdk/typescript`), Python SDK (`sdk/python`) | `exec` (JSONL) or `app-server` JSON‑RPC |
| **Cloud (Codex Web)** | `codex cloud` | chatgpt.com/codex | `cloud-tasks` → ChatGPT backend API |

So "the desktop application" = a thin native shell that connects to the **`app-server`** (a JSON‑RPC server in `codex-rs/app-server`), which orchestrates the **`core`** engine. The IPC layer that a desktop/IDE app uses is therefore the **app-server JSON‑RPC protocol**, documented in §5.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            FRONTENDS / CLIENTS                             │
│  TUI (ratatui)   Native Desktop App   IDE ext (VSCode/JetBrains)   SDKs    │
└───────┬───────────────┬──────────────────────┬───────────────────┬────────┘
        │ in-proc        │ stdio/UDS/WS          │ stdio/UDS/WS       │ spawn
        ▼               ▼                       ▼                   ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  app-server (JSON-RPC 2.0)  ── transport: stdio | unix socket | websocket    │
│  message_processor → request_processors/*  ⇄  thread_state / outgoing_message│
└───────────────────────────────┬──────────────────────────────────────────────┘
                                 ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  core  (the agent engine):  ThreadManager → Codex/Session → Task             │
│  Agent loop ⇄ ModelClient (Responses API) ⇄ ToolRouter/ToolOrchestrator      │
│  Tools: shell, apply_patch, unified_exec, MCP, plan, multi-agent, web_search │
│  Guardian (auto-approval)   Hooks   Skills   Plugins   Compaction            │
└───────┬───────────────┬───────────────┬───────────────┬─────────────────────┘
        ▼               ▼               ▼               ▼
  Sandboxing      Model Providers   Persistence     Observability
 (bwrap/landlock/  (OpenAI/Bedrock/  (rollout JSONL  (otel/analytics/
  seatbelt/win)     ollama/lmstudio)  + sqlite state) feedback)
```

**Key architectural patterns:** event-driven streaming (async channels), actor-like
submission loop, immutable per-turn `TurnContext`, JSON-RPC request/response
correlation, scoped concurrency serialization, event-sourced persistence (rollout),
sandbox policy transformation, centralized approval orchestration, multi-call binary
dispatch (`arg0`), and compile-time protocol→TypeScript/JSON-Schema generation.

---

## 1. Repository Folder Responsibility Map

| Path | Responsibility |
|---|---|
| `codex-rs/` | The Rust Cargo workspace — **all engine, server, TUI, sandbox, and provider code** (~92 crates). Built with both Cargo and Bazel. |
| `codex-cli/` | npm package `@openai/codex`. `bin/codex.js` detects platform/arch and execs the bundled native Rust binary. Scripts to build npm archive + firewall init. |
| `sdk/typescript/` | TypeScript SDK (`@openai/codex-sdk`). Spawns `codex exec` and parses JSONL events. |
| `sdk/python/` | Python SDK (`openai_codex`). Drives the `app-server` JSON-RPC **v2** protocol via subprocess; generated types from the protocol schema. |
| `sdk/python-runtime/` | Packages the codex binary inside a Python wheel (`codex_cli_bin`). |
| `docs/` | Documentation stubs/links (config, sandbox, skills, slash commands, auth, exec). |
| `tools/` | Repo tooling, e.g. `argument-comment-lint` (Bazel aspect + Rust driver). |
| `scripts/` | Packaging/build/test helpers (npm staging, blob-size check, mock WS server, etc.). |
| `third_party/` | Vendored third-party code. |
| `patches/` | pnpm patch files for JS deps. |
| `.codex/` | Repo-local Codex config: **skills** (`babysit-pr`, `code-review*`, `codex-bug`, etc.), environments. |
| `.devcontainer/` | Dev container + **firewall init** (`init-firewall.sh`) + secure variant. |
| `.github/` | CI workflows (Bazel, Rust CI, release, code-sign), issue/PR templates, Codex bot labels. |
| Root build files | `MODULE.bazel`/`BUILD.bazel`/`defs.bzl`/`.bazelrc` (Bazel), `flake.nix` (Nix), `justfile` (dev tasks), `pnpm-workspace.yaml`, `package.json`, `cliff.toml` (changelog). |

### `codex-rs` crate inventory (92 crates) by domain

| Domain | Crates |
|---|---|
| **Entry / dispatch** | `cli`, `arg0` |
| **Engine** | `core`, `core-api`, `core-plugins`, `core-skills`, `protocol` |
| **App-server (IPC)** | `app-server`, `app-server-protocol`, `app-server-transport`, `app-server-client`, `app-server-daemon`, `app-server-test-client` |
| **Frontends / modes** | `tui`, `exec`, `exec-server`, `mcp-server`, `cloud-tasks`, `cloud-tasks-client`, `cloud-tasks-mock-client`, `cloud-requirements` |
| **Model / LLM** | `model-provider`, `model-provider-info`, `models-manager`, `codex-api`, `codex-client`, `backend-client`, `codex-backend-openapi-models`, `ollama`, `lmstudio`, `aws-auth`, `responses-api-proxy`, `realtime-webrtc` |
| **Tools / extensibility** | `tools`, `code-mode`, `codex-mcp`, `rmcp-client`, `plugin`, `skills`, `connectors`, `hooks`, `collaboration-mode-templates`, `ext/{extension-api,goal,guardian,image-generation,memories,web-search}` |
| **Sandbox / security** | `sandboxing`, `linux-sandbox`, `bwrap`, `windows-sandbox-rs`, `execpolicy`, `execpolicy-legacy`, `process-hardening`, `network-proxy`, `shell-escalation`, `shell-command`, `file-system`, `secrets`, `agent-identity` |
| **Auth** | `login`, `chatgpt`, `keyring-store` |
| **Persistence / state** | `state`, `thread-store`, `rollout`, `rollout-trace`, `message-history`, `memories/{read,write}`, `agent-graph-store`, `external-agent-sessions`, `external-agent-migration` |
| **Config** | `config`, `features`, `install-context` |
| **Observability** | `otel`, `analytics`, `feedback`, `response-debug-context` |
| **Utilities** | `async-utils`, `ansi-escape`, `apply-patch`, `file-search`, `file-watcher`, `git-utils`, `terminal-detection`, `uds`, `stdio-to-uds`, `network-proxy`, `v8-poc`, `models-manager`, `test-binary-support`, `thread-manager-sample` |

---

## 2. Application Startup Lifecycle

```
$ codex [args]
  │
  ▼
codex-cli/bin/codex.js        (only when installed via npm)
  • detect platform+arch → resolve vendor/<triple>/bin/codex
  • set CODEX_MANAGED_BY_NPM, CODEX_MANAGED_PACKAGE_ROOT
  • exec native binary, forward SIGINT/SIGTERM/SIGHUP, mirror exit
  │
  ▼
cli/src/main.rs :: main()
  • arg0_dispatch_or_else(...)  ← arg0 crate
      – if argv[0] == codex-linux-sandbox → codex_linux_sandbox::run_main (never returns)
      – if argv[0] == apply_patch         → codex_apply_patch::main
      – if argv[1] == CODEX_FS_HELPER_ARG1 → exec_server fs helper
      – load ~/.codex/.env, build tokio multi-thread runtime, set PATH alias
  │
  ▼
Subcommand dispatch (clap Subcommand enum)
  • (none)/resume/fork → run_interactive_tui()  → codex_tui
  • exec/review        → codex_exec::run_main
  • app-server         → codex_app_server::run_main_with_transport_options
  • app                → desktop_app::run_app_open_or_install (mac/windows)
  • login/logout/mcp/plugin/cloud/doctor/sandbox/debug/...  → respective crate
  │
  ▼ (TUI path)
tui/src/lib.rs :: run_ratatui_app()
  • load Config (layered TOML), init State DB
  • choose app-server: Embedded (in-proc) | LocalDaemon (UDS) | Remote (ws://)
  • AppServerSession.bootstrap(): account, models, default model
  • thread_start() → ThreadManager → Codex::spawn() (core)
  • enter App::run() async event loop (tokio::select! on UI events,
    AppEvent bus, and app-server notifications/requests)
```

**Core spawn (engine init):** `ThreadManager::start_thread()` → `Codex::spawn()` constructs a `Session` (with `SessionServices`, `Arc<RwLock<SessionState>>`, event `tx_event`/`rx_event` channels) and starts the **submission loop** (`session/handlers.rs::submission_loop`) which consumes `Op` submissions and dispatches `Task`s.

---

## 3. Runtime Communication Flow (a full user turn, end-to-end)

```
User types prompt in TUI composer (bottom_pane/chat_composer.rs)
  │  AppEvent::SubmitPrompt(text)  (Elm-like message bus)
  ▼
App loop → AppServerSession.turn_start(items, cwd, model, …)        [JSON-RPC: turn/start]
  ▼
app-server: TurnRequestProcessor → ThreadManager → Session
  │  Op::UserTurnInput enqueued on submission loop
  ▼
core: build TurnContext (model, permissions, sandbox, tools registry)
       spawn RegularTask (tasks/regular.rs)
  ▼
core: Prompt built (base instructions + skills + plugins + history + user msg)
      ModelClientSession.stream()  → Responses API (HTTP SSE or WebSocket)
  ▼
Streamed ResponseEvents:
  • OutputTextDelta  → EventMsg::AgentMessageDelta → app-server ServerNotification
        item/agentMessage/delta → TUI appends to active HistoryCell (live render)
  • OutputItemDone (tool call) → ToolRouter.route() → handler
  ▼
Tool call (e.g. shell): ToolOrchestrator
  • execpolicy check + approval policy
  • if approval needed → ServerRequest item/commandExecution/requestApproval
        → TUI ApprovalOverlay → user decision → JSON-RPC response → core resumes
  • Guardian (optional auto-approval review session) may auto-decide
  • select SandboxType (bwrap/landlock/seatbelt/windows/none)
  • execute via runtime; stream CommandExecOutputDelta notifications → TUI
  ▼
Tool output fed back to model → loop until no more tool calls / stop
  ▼
TurnCompleted notification (token usage, diff, plan)
  • persist events to rollout JSONL (~/.codex/sessions/…) + sqlite state index
  • TUI commits active cell to history, updates status line
```

**Two streaming substrates:**
1. **core → app-server**: `codex_protocol::protocol::EventMsg` over async channel; a per-thread **listener task** maps `EventMsg` → protocol `ServerNotification`.
2. **app-server → client**: JSON-RPC notifications (one-way) and server requests (awaiting response, used for approvals/elicitation/user-input).

**Model wire protocol:** OpenAI **Responses API** (`/v1/responses`), streamed via **SSE** with a **WebSocket** fast-path (prewarm with `generate:false`, reuses `previous_response_id`, sticky `x-codex-turn-state` header). Chat Completions wire API has been removed; `WireApi::Responses` is the only variant.

---

## 4. Feature Dependency Graph (crate-level)

```
                         cli  (binary entry)
       ┌──────────────┬───────────────┬──────────────┬─────────────┐
       ▼              ▼               ▼              ▼             ▼
      tui           exec          app-server      cloud-tasks    login
       │              │            (+daemon,         │            │
       │              │             transport)       ▼            ▼
       └──────┬───────┴───────┬───────┘         cloud-tasks-   chatgpt,
              ▼               ▼                    client      keyring-store
        app-server-client  app-server-protocol
              │                   │ (schema → TS/JSON for SDKs)
              ▼                   ▼
            ┌─────────────────────────────────┐
            │             core                │
            └──┬───────┬───────┬───────┬───────┘
               ▼       ▼       ▼       ▼
          model-    tools/   sandboxing  rollout/state/
          provider  mcp/     +linux-     thread-store/
          +api      plugin/  sandbox+    message-history
          +models-  skills/  bwrap+      +memories
          manager   hooks    win-sandbox
               │     +code-   +execpolicy
               ▼     mode     +network-proxy
          codex-api,         +process-hardening
          backend-client,
          ollama, lmstudio,
          aws-auth, realtime-webrtc
```

---

## 5. The App-Server IPC Protocol (Desktop/IDE backend)

**Crate:** `app-server` (server), `app-server-protocol` (typed messages + schema gen),
`app-server-transport` (wire), `app-server-client` (client), `app-server-daemon` (lifecycle/auto-update).

### 5.1 Transport
- Protocol: **JSON-RPC 2.0** (simplified; `jsonrpc_lite.rs`). Messages: `JSONRPCRequest{id,method,params,trace}`, `JSONRPCNotification`, `JSONRPCResponse{id,result}`, `JSONRPCError`.
- Transports (`transport/`): `stdio://` (default, IDE), `unix://[PATH]` (local IPC), `ws://IP:PORT` (remote). Control socket (`~/.codex/app-server-control.sock`) for daemon lifecycle.
- `trace` field carries **W3C trace context** for OpenTelemetry.

### 5.2 Protocol surface (defined via macros in `protocol/common.rs`; schema in `schema/json/*.json`)

**Client → Server requests** (selected, grouped):
- *Thread lifecycle:* `thread/start`, `thread/resume`, `thread/fork`, `thread/archive`, `thread/unarchive`, `thread/name/set`, `thread/list`, `thread/read`, `thread/turns/list`, `thread/inject_items`, `thread/compact/start`, `thread/goal/{set,get,clear}`.
- *Turn:* `turn/start`, `turn/steer`, `turn/interrupt`.
- *Approvals/permissions:* `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `item/permissions/requestApproval` (legacy `ExecCommandApproval`/`ApplyPatchApproval`).
- *Filesystem:* `fs/{readFile,writeFile,createDirectory,getMetadata,readDirectory,remove,copy,watch,unwatch}`.
- *Skills/plugins/hooks:* `skills/{list,extraRootsSet,configWrite}`, `hooks/list`, `plugin/{list,read,install,uninstall}`, `marketplace/{add,remove,upgrade}`.
- *Account/config:* `account/status`, `account/login`, `account/logout`, `account/get`, `account/rateLimits`, `collaboration-mode/list`, model list/config writes.
- *Search/tools:* `FuzzyFileSearch*`, `mcpServer/toolCall`, `mcpServer/oauthLogin`, `mcpServer/status/list`.
- *Env/process/terminal:* `environment/add`, `process/*` (experimental), `command/exec`.
- *Realtime (experimental):* `thread/realtime/{start,appendAudio,appendText,stop}`.

**Server → Client requests** (awaiting response): `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `item/permissions/requestApproval`, `item/tool/requestUserInput`, `mcpServer/elicitation/request`, `item/tool/call` (DynamicToolCall), `account/chatgptAuthTokens/refresh`, `attestation/generate`.

**Server → Client notifications** (one-way stream): `thread/{started,status/changed,archived,closed,name/updated,goal/*,tokenUsage/updated,settings/updated}`, `turn/{started,completed,diff/updated,plan/updated}`, `item/{started,completed}`, `hook/{started,completed}`, deltas: `item/agentMessage/delta`, `command/exec/outputDelta`, `item/commandExecution/outputDelta`, `item/fileChange/{outputDelta,patchUpdated}`, `item/reasoning/{summaryTextDelta,summaryPartAdded,textDelta}`, `item/mcpToolCall/progress`, plus `account/{updated,rateLimits/updated}`, `mcpServer/*`, `fs/changed`, `error`, `warning`, `guardianWarning`, `deprecationNotice`, `configWarning`.

### 5.3 Server internals (`app-server/src`)
- `message_processor.rs` — central dispatcher; owns all request processors + `RequestSerializationQueues`.
- `request_processors/` — one processor per domain: `thread_processor.rs` (4.2k LOC), `turn_processor.rs`, `command_exec_processor.rs`, `fs_processor.rs`, `search.rs`, `mcp_processor.rs`, `config_processor.rs`, `account_processor.rs`, `git_processor.rs`, `feedback_processor.rs`, `marketplace_processor.rs`, `plugin_processor.rs`, `environment_processor.rs`, `apps_processor.rs`, `catalog_processor.rs`, `external_agent_config_processor.rs`, `windows_sandbox_processor.rs`, `process_exec_processor.rs`, `remote_control_processor.rs`, `initialize_processor.rs`.
- `outgoing_message.rs` — `OutgoingMessageSender`: server-request id allocation, request/response correlation (oneshot), broadcast vs. thread-scoped delivery.
- `thread_state.rs` / `thread_status.rs` — per-thread state machine (pending interrupts/rollbacks, listener task) + load-state gating.
- `request_serialization.rs` — **scoped concurrency**: thread-scoped requests serialize per `ThreadId`; global/fs/account run concurrently.
- `config_manager.rs`, `fs_watch.rs`, `skills_watcher.rs`, `mcp_refresh.rs`, `dynamic_tools.rs`, `attestation.rs`, `bespoke_event_handling.rs`, `connection_rpc_gate.rs` (graceful shutdown), `in_process.rs` (embedded client mode).

### 5.4 Daemon & auto-update (`app-server-daemon`)
- `LifecycleCommand::{Start,Restart,Stop,Version}`; managed install to `~/.codex/managed-codex/`.
- `update_loop.rs` — separate updater process polls (~60 min) for a new managed binary; restarts app-server via control socket (`IfVersionChanged`/`Always`).
- `remote_control_client.rs` — optional encrypted WebSocket tunneling for remote IDE connections; `settings.rs` persists daemon settings.

### 5.5 SDK type generation
- `app-server-protocol` `bin/export.rs` generates **TypeScript** (`v2/*.ts`) and **JSON Schemas** from the Rust protocol types (`schemars` + `ts-rs`). `#[experimental]` fields are stripped from public exports. Drives the Python SDK's `generated/` and IDE typings. (`just write-app-server-schema`.)

---

## 6. The Core Engine (`codex-rs/core`)

**Role:** central agent orchestration — session lifecycle, turn processing, model
calls, tool execution + sandboxing + approvals, context/compaction, and persistence.

### 6.1 Key types
| Type | File | Purpose |
|---|---|---|
| `Codex` | `session/mod.rs` | Public session handle: `tx_sub` (Submissions), `rx_event` (Events), `agent_status` watch. |
| `Session` | `session/session.rs` | Internal state container: `conversation_id`, `services`, `state: Arc<RwLock<SessionState>>`, `agent_control`. |
| `ThreadManager` | `thread_manager.rs` | Factory: `start_thread`, `fork_thread`, `resume_thread`, `shutdown`. |
| `CodexThread` | `codex_thread.rs` | Live thread + config snapshot + runtime overrides. |
| `TurnContext` | `session/turn_context.rs` | Immutable per-turn snapshot (model, permissions, env, tools). |
| `AgentControl` | `agent/control.rs` | Multi-agent: `spawn_agent`, `send_message`, `wait_for_agent`. |
| `ModelClient` / `ModelClientSession` | `client.rs` | Session/turn-scoped Responses API client (WS+SSE). |
| `ToolRouter` / `ToolOrchestrator` | `tools/router.rs`, `tools/orchestrator.rs` | Tool dispatch + approval/sandbox orchestration. |
| `Config` | `config/mod.rs` | Layered immutable config (`config.toml`). |

### 6.2 Submission loop & tasks
The `submission_loop` consumes `Op` (e.g. `UserTurnInput`, `Interrupt`, `Shutdown`) and
spawns one of: **RegularTask** (agent turn), **ReviewTask** (guardian/code review),
**CompactTask** (context compaction), **UserShellCommandTask** (direct shell). Each turn
is a `tokio::spawn`ed task driving the model↔tools loop; cancellation via
`CancellationToken`.

### 6.3 Tool handlers (`core/src/tools/handlers/`)
`shell`, `unified_exec` (PTY/interactive, reusable processes), `apply_patch`, `plan`,
`goal`, `agent_jobs` (CSV batch), `multi_agents`/`multi_agents_v2`, `mcp` + `mcp_resource*`,
`request_permissions`, `request_user_input`, `request_plugin_install`, `tool_search`,
`view_image`, `extension_tools`, `dynamic`, plus `code_mode` execute/wait.

### 6.4 Guardian (auto-approval), Hooks, Compaction
- **Guardian** (`guardian/`, `ext/guardian`): spawns a separate review session that scores risky on-request actions (`GuardianAssessment{risk_level,outcome}`) to auto-approve/deny/escalate (≈90s timeout).
- **Hooks** (`hook_runtime.rs` + `hooks` crate): lifecycle events `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PreCompact`, `Stop`, `PermissionRequest`; configured in `config.toml`/`requirements.toml` (managed-only mode via `allow_managed_hooks_only`).
- **Compaction** (`compact.rs`, `compact_remote_v2.rs`): summarizes history when nearing the context window (auto) or on `/compact` (manual); remote variant offloads to `/responses/compact`.

---

## 7. The TUI Frontend (`codex-rs/tui`, ratatui)

- **Architecture:** Elm-like message bus. `App::run()` async loop `tokio::select!`s over UI events (crossterm), the `AppEvent` channel, and app-server notifications/requests. Widgets emit `AppEvent` via `AppEventSender`; the loop mutates state and redraws.
- **Component hierarchy:** `App` → `ChatWidget` (transcript of `HistoryCell`s + streaming active cell) + `BottomPane` (composer + LIFO `view_stack` of modal `BottomPaneView`s).
- **`ChatComposer`** (largest file, ~11k LOC): rich input — Vim mode, reverse history search (Ctrl+R), `@file`/`$skill` mentions, paste bursting, external editor (Ctrl+E), slash-command preview. Backed by low-level `textarea.rs`.
- **Backend link:** `app_server_session.rs` wraps `AppServerClient` (InProcess | Remote) and exposes typed JSON-RPC calls (`thread_start/resume/fork`, `turn_start/interrupt`, `thread_read`, config/account/model APIs).
- **Modals/`BottomPaneView`:** `ApprovalOverlay`, `McpServerElicitationOverlay`, `RequestUserInputOverlay`, `ListSelectionView`, keymap setup, feedback, hooks browser, resume/fork picker.
- **Rendering:** `markdown_render.rs` (pulldown-cmark + syntect highlighting), `diff_render.rs` (unified diff with gutters), terminal hyperlinks.

---

## 8. Model / LLM Integration

- **`model-provider-info`** — `ModelProviderInfo{name,base_url,env_key,wire_api,http_headers,retries,timeouts,requires_openai_auth,supports_websockets,aws,…}`. Built-ins: `openai`, `amazon-bedrock`, `ollama` (11434), `lmstudio` (1234). `WireApi::Responses` only.
- **`model-provider`** — runtime `ModelProvider` trait + `ConfiguredModelProvider` / `AmazonBedrockModelProvider` (SigV4 via `aws-auth`, Mantle endpoint).
- **`models-manager`** — `ModelsManager` (Online/Offline/OnlineIfUncached refresh), bundled `models.json` fallback, per-model metadata (context window, reasoning levels, truncation policy), collaboration modes.
- **`codex-api`** — endpoint clients: `ResponsesClient`, `ResponsesWebsocketClient`, `RealtimeCallClient`, `ModelsClient`, `CompactClient`, `MemoriesClient`, `SearchClient`, `ImagesClient`; SSE parsing (`spawn_response_stream` → `ResponseEvent`), `AuthProvider` trait (Bearer / command / SigV4 / ChatGPT session).
- **`codex-client`** — low-level transport/byte-stream + retry. **`backend-client`** — internal ChatGPT backend (`/api/codex/*`, `/wham/*`): tasks, rate limits, credits. **`codex-backend-openapi-models`** — generated structs.
- **Local models:** `ollama`, `lmstudio` (OpenAI-compatible, no auth). **`responses-api-proxy`** — local dev proxy forwarding `/v1/responses` (auth from stdin, optional dump dir). **`realtime-webrtc`** — macOS voice via WebRTC (SDP offer, audio levels).
- **Retry:** HTTP 4 attempts (200ms base backoff); stream reconnect 5; idle timeout 300s; WS connect 15s.

---

## 9. Sandboxing & Security

| Platform | Mechanism | Crate / files |
|---|---|---|
| **Linux** | **Landlock** (FS LSM) + **seccomp**, and/or **Bubblewrap (bwrap)** namespace isolation; bundled or system bwrap | `linux-sandbox` (`landlock.rs`, `bwrap.rs`, `bundled_bwrap.rs`, `launcher.rs`, `linux_run_main.rs`), `sandboxing` (`landlock.rs`, `bwrap.rs`), `bwrap` |
| **macOS** | **Seatbelt** (`sandbox-exec`) with `.sbpl` profiles | `sandboxing` (`seatbelt.rs`, `seatbelt_base_policy.sbpl`, `seatbelt_network_policy.sbpl`, `restricted_read_only_platform_defaults.sbpl`) |
| **Windows** | Restricted-token / **AppContainer-style ACLs**, deny-read ACLs, ConPTY | `windows-sandbox-rs` (`acl.rs`, `allow.rs`, `deny_read_acl.rs`, `cap.rs`, `desktop.rs`, `conpty/`), `core/src/windows_sandbox*.rs` |

- **`sandboxing/manager.rs` + `policy_transforms.rs`** — convert an approval/permission policy into a concrete platform `SandboxType` and command transform.
- **Approval modes:** read-only, workspace-write, danger-full-access; on-request / on-failure / never. Escalation via **`shell-escalation`** (`codex-execve-wrapper`, sudo). Approval results cached per-turn by action hash; sandbox-denied commands can retry unsandboxed without re-prompting.
- **`execpolicy`** — **Starlark** prefix-based rules engine (`parser.rs`, `rule.rs`, `policy.rs`, `decision.rs`, `amend.rs`). Rule forms: `prefix_rule(pattern=[…], decision="allow|prompt|forbidden", match=…, not_match=…)`, `host_executable(name, paths=[…])`, `network_rule(...)`. Decision = strictest match; `prompt` rejected under `approval_policy="never"`. `execpolicy-legacy` (`define_program`) is the prior engine, kept for compatibility.
- **Linux specifics:** `codex-linux-sandbox` re-execs itself with `--apply-seccomp-then-exec`; outer stage builds **bwrap** argv (`--ro-bind / /`, `--bind` writable roots, `.git`/`.codex`/`.agents` kept `--ro-bind`), inner stage applies `PR_SET_NO_NEW_PRIVS` + **seccomp** (blocks `ptrace`, `process_vm_*`, `io_uring_*`, and network syscalls per `BwrapNetworkMode::{Isolated,ProxyOnly}`). Legacy fallback uses **Landlock ABI v5**. Unreadable globs masked via `/dev/null` mounts; synthetic mounts + `ProtectedCreateMonitor` guard protected paths.
- **macOS:** `sandbox-exec` with embedded `.sbpl` (base + network + restricted-read-only defaults); proxy loopback ports extracted from `HTTP(S)_PROXY`/`ALL_PROXY`.
- **Windows:** restricted-token modes (`ReadOnlyCapability`/`WritableRootsCapability`), DACL deny/allow ACEs + **capability SIDs** for writable roots, **WFP** network block, ConPTY, optional private desktop / elevated runner.
- **Network egress control** — **`network-proxy`**: a **MITM HTTP (`127.0.0.1:3128`) + SOCKS5 (`:8081`) proxy** with on-the-fly cert generation (`certs.rs`, `mitm.rs`, `socks5.rs`, `connect_policy.rs`, `network_policy.rs`). Domain allow/deny lists, `full`/`limited` (GET/HEAD/OPTIONS) modes, MITM header hooks (e.g. `strip_auth`); rejections carry `x-proxy-error` + OTEL `codex.network_proxy.policy_decision`.
- **`process-hardening`** (`#[ctor]` pre-main) — `PR_SET_DUMPABLE=0` (Linux), `PT_DENY_ATTACH` (macOS), `RLIMIT_CORE=0`, clears `LD_*`/`DYLD_*` env vars.
- **`responses-api-proxy`** hardening: API key read from **stdin only**, held in `mlock`ed/`zeroize`d memory, `HeaderValue::set_sensitive`, only `POST /v1/responses` allowed.
- **Sandbox env signals:** `CODEX_SANDBOX_NETWORK_DISABLED=1`, `CODEX_SANDBOX=seatbelt`, `CODEX_LINUX_SANDBOX_ARG0`. Approval modes: `UnlessTrusted`, `OnRequest` (default), `Granular{sandbox_approval,rules,skill_approval,request_permissions,mcp_elicitations}`, `Never` (legacy `OnFailure`).
- **`secrets`** — secret sanitization (`sanitizer.rs`) to redact credentials from logs/output. **`keyring-store`** — OS credential storage for auth. **`agent-identity`** — JWT-based agent identity (JWKS validation).
- **`file-system`** — sandbox-aware FS abstraction used by exec/apply-patch.

---

## 10. Tools / MCP / Plugins / Skills / Connectors / Hooks

- **`tools`** — tool schema & registry: `tool_spec.rs`, `tool_definition.rs`, `tool_config.rs`, `tool_discovery.rs`, `tool_executor.rs`, `dynamic_tool.rs`, `mcp_tool.rs`, `responses_api.rs`, `json_schema.rs`, `code_mode.rs`, `request_plugin_install.rs`. Core abstractions: `ToolExecutor<ToolInvocation>` trait (`tool_name`/`spec`/`handle`/`supports_parallel_tool_calls`), `CoreToolRuntime` (adds `search_info`, `telemetry_tags`, pre/post-tool-use hook payloads, diff consumers), `ToolName{namespace,name}` (e.g. `mcp__github__create_issue`), `ToolSpec::{Namespace,Function}`. Dispatch: model `FunctionCall` → `router.rs` → `registry.rs` lookup → handler → `FunctionCallOutput`. Tool search uses BM25 fuzzy matching.
- **MCP (Model Context Protocol):**
  - **`codex-mcp`** — orchestrates external MCP servers (config in `config.toml`, launched over stdio/http).
  - **`rmcp-client`** — the Rust MCP SDK client: `oauth.rs`, `elicitation_client_service.rs`, `executor_process_transport.rs`, `in_process_transport.rs`, `http_client_adapter`.
  - **`mcp-server`** — runs Codex **as** an MCP server (`codex mcp-server`): `codex_tool_runner.rs`, `exec_approval.rs`, `patch_approval.rs`, `message_processor.rs`.
- **`plugin` + `core-plugins`** — plugin system: `manifest.rs`, `loader.rs`, `manager.rs`, `store.rs`, `marketplace*.rs` (add/remove/upgrade), `remote*.rs` + `startup_remote_sync.rs` (remote bundle sync), `toggles.rs`. CLI: `cli/src/plugin_cmd.rs`, `marketplace_cmd.rs`.
- **`skills` + `core-skills`** — authored `SKILL.md` skills: `loader.rs`, `manager.rs`, `injection.rs` (prompt injection), `invocation_utils.rs` (implicit detection), `render.rs`, `remote.rs`, `system.rs`. Repo ships skills in `.codex/skills/`.
- **`connectors`** — discoverable "apps"/connectors with accessibility filtering + directory cache.
- **`code-mode`** — code-execution tool runtime (`runtime/`, `service.rs`, `response.rs`); `v8-poc` is a V8 JS-engine PoC for executing model-authored code.
- **`hooks`** — lifecycle hook engine (events in `events/`: `session_start`, `user_prompt_submit`, `pre_tool_use`, `post_tool_use`, `stop`, `compact`, `permission_request`), `registry.rs`, `engine/`, `schema.rs`, `config_rules.rs`.
- **`ext/` extensions** (via `extension-api`): `goal` (goal accounting/events), `guardian` (auto-approval), `image-generation` (image backend), `memories` (long-term memory), `web-search` (web search + history). The **`extension-api`** decouples extensions from `core` via contributor traits — `ToolContributor`, `ContextContributor`, `ThreadLifecycleContributor`, `ToolLifecycleContributor`, `TurnLifecycleContributor`, `TokenUsageContributor`, `ApprovalReviewContributor` — plus capabilities `AgentSpawner`, `ResponseItemInjector`, `TurnItemEmitter`, `ExtensionEventSink`, assembled in an `ExtensionRegistry`.

---

## 11. Persistence, State & Observability

- **`rollout`** — event-sourced session recording: `recorder.rs` writes JSONL to `~/.codex/sessions/<date>/<thread_id>.jsonl`; `list.rs`/`search.rs`/`session_index.rs`/`metadata.rs` for listing; `policy.rs`. `rollout-trace` adds reduced-state replay (`replay_bundle`).
- **`state`** — SQLite **State DB** (`log_db.rs`, `migrations.rs`, `runtime.rs`, `paths.rs`, `telemetry.rs`, `audit.rs`): thread metadata index, log DB, memories DB. `StateRuntime` is the handle (`cli` exposes `state-db-recovery`).
- **`thread-store`** — abstract thread persistence trait; **`message-history`** — `~/.codex/history.jsonl` cross-session input history.
- **`memories/{read,write}` + `ext/memories`** — long-term memory: write pipeline (`phase1.rs`/`phase2.rs`, `consolidation`), read with citations, workspace-scoped storage + guards.
- **`agent-graph-store`** — local graph store (`store.rs`, `local.rs`, `types.rs`) for agent relationships. **`external-agent-sessions`/`-migration`** — import/migrate other agents' sessions.
- **Observability:** **`otel`** — OpenTelemetry traces/metrics with `OtelExporter::{Otlp,Statsig}`, W3C trace-context propagation, `metrics/names.rs` (`codex.db.*`, `codex.sqlite.init.*`, `codex.turn.*`), `session_telemetry.rs`, runtime metrics; **`analytics`** — batched event client (`SkillInvocation`, `ThreadInitialized`, `GuardianReview`, hook runs, `Compaction`, turn/command/file/MCP/web-search/image events, `AcceptedLineFingerprints`, plugin lifecycle) + `facts.rs`/`reducer.rs`; **`feedback`** — ring-buffer log capture → **Sentry** upload with doctor report/diagnostics, classification tags; **`response-debug-context`** — debug capture.
- **Runtime context:** **`terminal-detection`** (identifies iTerm2/Ghostty/WezTerm/kitty/VSCode/WindowsTerminal + tmux/zellij for UA + features), **`install-context`** (`InstallMethod::{Standalone,Npm,Bun,Brew,Other}` from exe path/env), **`features`** (bitmask flag registry with `UnderDevelopment`/`Experimental`/`Stable`/`Deprecated` stages, surfaced in `/experimental`).

**`~/.codex/` on-disk layout (verified):**
```
~/.codex/
├── config.toml / *.config.toml (profiles) / requirements.toml
├── auth.json            # tokens / API key (0600; or OS keyring)
├── .env                 # loaded by arg0 before runtime starts
├── history.jsonl        # global input history (append-only, O_APPEND, advisory locks, size cap)
├── sessions/YYYY/MM/DD/rollout-<ISO8601>-<uuid>.jsonl   # rollout event log
├── archived_sessions/   # closed/archived threads
├── state_5.sqlite       # thread metadata, agent jobs, backfill (StateRuntime; WAL)
├── logs_2.sqlite        # structured logs (LogDb)
├── goals_1.sqlite       # thread goals + accounting
├── memories_1.sqlite    # long-term memory index
├── memories/            # git-backed memory workspace (raw_memories.md, MEMORY.md, rollout_summaries/)
├── managed-codex/       # daemon auto-update binary
├── packages/standalone/releases/<version>/  # install-context managed release (bin/, codex-resources/)
├── app-server-control.sock
└── plugins/ , skills/ , logs/
```
**Storage env vars:** `CODEX_SQLITE_HOME` (DB dir override), `CODEX_ROLLOUT_TRACE_ROOT_ENV` (trace bundles), `RUST_LOG`. Stores: rollout JSONL (`recorder.rs`), sqlx/SQLite WAL `StateRuntime`, `thread-store` (`LocalThreadStore`: list/read/append/archive/search), `message-history`, two-phase **memories** pipeline (per-rollout extract → global consolidation under git baseline), `agent-graph-store` (parent→child spawn edges), `keyring-store`, `agent-identity` (Ed25519 keys + JWT/JWKS).

---

## 12. CLI Subcommand Reference (`cli/src/main.rs`)

| Subcommand | Dispatches to | Purpose |
|---|---|---|
| *(none)* / `resume` / `fork` | `run_interactive_tui` (`tui`) | Interactive TUI; resume/fork a session |
| `exec` (`e`) / `review` | `codex_exec::run_main` | Headless run / non-interactive code review |
| `login` / `logout` | `codex_cli::run_login_*` / `run_logout` | ChatGPT OAuth, device-code, API key, access-token; logout |
| `mcp` | `mcp_cmd` | Manage external MCP servers |
| `mcp-server` | `codex_mcp_server::run_main` | Run Codex as an MCP server (stdio) |
| `plugin` / *(marketplace)* | `plugin_cmd` / `marketplace_cmd` | Manage plugins / marketplace |
| `app-server` | `codex_app_server::run_main_with_transport_options` | Run the JSON-RPC app-server (+ daemon/proxy/schema gen) |
| `remote-control` | `remote_control_cmd` | Manage app-server daemon w/ remote control |
| `app` | `desktop_app::run_app_open_or_install` | Install/open the **native Desktop App** (mac/win) |
| `cloud` (`cloud-tasks`) | `codex_cloud_tasks::run_main` | Browse/apply Codex Cloud (Web) tasks |
| `doctor` | `doctor::run_doctor` | Diagnostics (git, runtime, system, updates, threads, background) |
| `sandbox` | `run_command_under_*` | Run a command inside the Codex sandbox |
| `apply` (`a`) | `codex_chatgpt::run_apply_command` | Apply latest diff via `git apply` |
| `archive` / `unarchive` | `codex_tui::run_session_archive_command` | Archive/unarchive sessions |
| `execpolicy` | `run_execpolicycheck` | Test execpolicy rules |
| `completion` | clap `generate` | Shell completions |
| `features` | enable/disable feature flags | Feature inspection |
| `debug` | models / app-server / prompt-input / trace-reduce / clear-memories | Internal debug tools |
| `responses-api-proxy` / `stdio-to-uds` / `exec-server` | respective crates | Internal services (proxy, relay, remote exec) |

**`arg0` multi-call dispatch:** one binary behaves as `codex`, `codex-linux-sandbox`,
`apply_patch`, `codex-execve-wrapper`, or the FS helper, based on `argv[0]`/`argv[1]`.

---

## 13. Authentication Flow

- **Modes (`CodexAuth`):** `ApiKey`, `Chatgpt` (OAuth, disk), `ChatgptAuthTokens` (in-memory), `AgentIdentity` (JWT).
- **Logins:** Sign in with ChatGPT (local OAuth server + browser, code→token at `auth.openai.com/oauth/token`), **device-code** (headless), API key from stdin, access token from stdin, Agent Identity JWT from `CODEX_ACCESS_TOKEN` (JWKS-validated).
- **Storage:** `~/.codex/auth.json` (`AuthDotJson`), 0600; optional **keyring** backend (service "Codex Auth"). **Refresh:** every ~8 min, or when <5 min to expiry; transient vs permanent failure handling; optional revoke on logout. The app-server refreshes ChatGPT tokens via a `ChatgptAuthTokensRefresh` server-request to the client.

---

## 14. The SDKs

**TypeScript (`sdk/typescript`):** `Codex` class → `startThread()`/`resumeThread()` → `Thread.run()`/`runStreamed()`. Internally `CodexExec` (`exec.ts`) spawns **`codex exec --experimental-json …`** (config overrides via `--config` TOML flags) and parses JSONL `ThreadEvent`s (`ItemStarted/Updated/Completed`, `TurnCompleted`, …). Sets `CODEX_INTERNAL_ORIGINATOR_OVERRIDE=codex_sdk_ts`. Binary resolved from `@openai/codex-<platform>-<arch>` packages. Exposes `ApprovalMode`, `SandboxMode`, `ModelReasoningEffort`, `WebSearchMode`, item types.

**Python (`sdk/python`):** `Codex`/`AsyncCodex` (`api.py`) spawn **`codex run --experimental-json`** and drive the **app-server JSON-RPC v2 protocol** over subprocess stdio via `MessageRouter` (`client.py`); types are **generated** from the protocol schema (`generated/v2_all.py`, `notification_registry.py`). Modules: `_run.py`, `_login.py` (`ChatgptLoginHandle`, `DeviceCodeLoginHandle`), `_inputs.py` (`TextInput`/`LocalImageInput`/`SkillInput`/`MentionInput`), `_approval_mode.py`, `_sandbox.py`, `retry.py`. `python-runtime` bundles the platform binary into a per-platform wheel (`hatch_build.py`, resolved via `importlib.resources`).

---

## 15. Complete Feature Matrix

> Per request: every button/menu/command/setting/IPC handler/background job/utility/hook/provider/store/service/API route/UI component is a **separate** row.

### 15.1 CLI commands & dispatch

| Feature | What it does | User-facing/internal | Main files | Related files | Key fns/types | UI components | IPC/desktop wiring | APIs/services | State/data flow | Deps | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `codex` (TUI launch) | Start interactive agent | User | `cli/src/main.rs`, `tui/src/lib.rs` | `app_server_session.rs` | `run_interactive_tui`, `App::run` | full TUI | embeds/connects app-server | core | event loop | tui,core | default subcommand |
| `codex exec` | Headless turn | User | `exec/src/lib.rs`,`cli.rs` | `exec_events.rs` | `run_main`, `EventProcessor*` | none/stdout | in-proc app-server-client | core | JSONL stream | exec | `--json`,`--output-schema` |
| `codex review` | Non-interactive review | User | `exec` | `core/review_*` | `ExecCommand::Review` | stdout | in-proc | core | — | exec | wraps exec |
| `codex login` | ChatGPT/API auth | User | `cli/src/login.rs` | `login`,`chatgpt` | `run_login_with_chatgpt` | browser | local OAuth server | auth.openai.com | writes auth.json | login | OAuth/device/apikey |
| `codex logout` | Clear creds | User | `cli/src/login.rs` | `login` | `run_logout` | — | — | revoke endpoint | deletes tokens | login | |
| `codex mcp` | Manage MCP servers | User | `cli/src/mcp_cmd.rs` | `codex-mcp` | `mcp_cli.run` | — | — | MCP | config.toml | codex-mcp | |
| `codex mcp-server` | Codex as MCP server | User | `mcp-server/src/lib.rs` | `codex_tool_runner.rs` | `run_main` | — | stdio JSON-RPC (MCP) | core | tool calls | mcp-server | |
| `codex plugin` | Manage plugins | User | `cli/src/plugin_cmd.rs` | `plugin` | `run_plugin_*` | — | — | marketplace | plugin store | plugin | |
| `codex app` | Install/open Desktop App | User | `cli/src/app_cmd.rs`,`desktop_app/` | `mac.rs`,`windows.rs` | `run_app_open_or_install` | native app | downloads+launches app | download URL | — | — | mac/win only |
| `codex app-server` | Run JSON-RPC server | User/internal | `app-server/src/main.rs` | transport | `run_main_with_transport_options` | — | stdio/uds/ws | core | RPC | app-server | desktop/IDE backend |
| `codex remote-control` | Daemon remote ctrl | User | `cli/src/remote_control_cmd.rs` | `app-server-daemon` | `run` | — | control socket | — | settings | daemon | |
| `codex cloud` | Cloud tasks | User | `cloud-tasks/src/lib.rs` | `cloud-tasks-client` | `run_main`,`create_task` | TUI list | — | chatgpt backend | task diffs | cloud-tasks | |
| `codex doctor` | Diagnostics | User | `cli/src/doctor.rs` + `doctor/*` | git/runtime/system/updates | `run_doctor` | report | — | update check | reads config | cli | |
| `codex sandbox` | Run under sandbox | User | `cli/src/debug_sandbox.rs`,`sandbox_setup.rs` | `linux-sandbox` etc | `run_command_under_*` | — | — | — | — | sandboxing | |
| `codex apply` | git apply latest diff | User | `chatgpt/apply_command.rs` | git-utils | `run_apply_command` | — | — | — | reads rollout | chatgpt | |
| `codex archive`/`unarchive` | (Un)archive session | User | `tui` session archive | rollout | `run_session_archive_command` | — | — | — | sqlite state | tui | |
| `codex execpolicy` | Test exec rules | Internal | `cli` execpolicy cmd | `execpolicy` | `run_execpolicycheck` | — | — | — | Starlark | execpolicy | |
| `codex completion` | Shell completion | User | `cli/src/main.rs` | — | clap `generate` | — | — | — | — | clap | |
| `codex features` | Feature flags | User | `cli` features | `features` | enable/disable | — | — | — | config | features | |
| `codex debug …` | Internal tools | Internal | `cli/src/main.rs` | — | `run_debug_*` | — | — | — | — | cli | models/trace/etc |
| `codex responses-api-proxy` | Local responses proxy | Internal | `responses-api-proxy` | dump.rs | `run_main` | — | HTTP | OpenAI | forwards | proxy | |
| `codex stdio-to-uds` | stdio↔UDS relay | Internal | `stdio-to-uds` | uds | `run` | — | UDS | — | — | uds | |
| `codex exec-server` | Remote exec service | Internal | `exec-server` | environment | exec server | — | ws/stdio/uds | — | process mgmt | exec-server | |
| `arg0 dispatch` | Multi-call binary | Internal | `arg0/src/lib.rs` | `cli/main.rs` | `arg0_dispatch_or_else` | — | — | — | PATH alias | arg0 | codex-linux-sandbox/apply_patch |

### 15.2 App-server IPC handlers (each = a feature row)

| IPC method | Handler file | Direction | What it does | Notifications emitted |
|---|---|---|---|---|
| `initialize` | `initialize_processor.rs` | C→S req | Handshake/capabilities | — |
| `thread/start` | `thread_processor.rs` | C→S req | Create thread | `thread/started` |
| `thread/resume` | `thread_processor.rs`,`thread_resume_redaction.rs` | C→S req | Resume + redact history | `thread/started` |
| `thread/fork` | `thread_processor.rs` | C→S req | Fork at snapshot | `thread/started` |
| `thread/archive`,`/unarchive` | `thread_processor.rs` | C→S req | Archive state | `thread/archived` |
| `thread/name/set` | `thread_processor.rs` | C→S req | Rename | `thread/name/updated` |
| `thread/list`,`/read`,`/turns/list` | `thread_processor.rs` | C→S req | Query threads/turns | — |
| `thread/inject_items` | `thread_processor.rs` | C→S req | Inject raw items | — |
| `thread/compact/start` | `turn_processor.rs` | C→S req | Manual compaction | `thread/compacted` |
| `thread/goal/{set,get,clear}` | `thread_goal_processor.rs` | C→S req | Goal CRUD | `thread/goal/updated` |
| `turn/start` | `turn_processor.rs` | C→S req | Start a turn | `turn/started`,deltas,`turn/completed` |
| `turn/steer` | `turn_processor.rs` | C→S req | Steer running turn | — |
| `turn/interrupt` | `turn_processor.rs` | C→S req | Cancel turn | — |
| `item/commandExecution/requestApproval` | `command_exec_processor.rs` | S→C req | Approve command | resolved |
| `item/fileChange/requestApproval` | `turn_processor.rs` | S→C req | Approve patch | resolved |
| `item/permissions/requestApproval` | `turn_processor.rs` | S→C req | Permission escalation | resolved |
| `item/tool/requestUserInput` | `dynamic_tools.rs` | S→C req | Ask user input | resolved |
| `mcpServer/elicitation/request` | `mcp_processor.rs` | S→C req | MCP prompts user | resolved |
| `item/tool/call` (DynamicToolCall) | `dynamic_tools.rs` | S→C req | Client-side tool | resolved |
| `account/chatgptAuthTokens/refresh` | `account_processor.rs` | S→C req | Refresh OAuth | — |
| `attestation/generate` | `attestation.rs` | S→C req | Attestation proof | — |
| `fs/{readFile,writeFile,createDirectory,getMetadata,readDirectory,remove,copy}` | `fs_processor.rs` | C→S req | File ops | — |
| `fs/watch`,`fs/unwatch` | `fs_watch.rs` | C→S req | Watch FS | `fs/changed` |
| `FuzzyFileSearch*` | `search.rs`,`fuzzy_file_search.rs` | C→S req | Fuzzy file search | session updated |
| `mcpServer/toolCall`,`/oauthLogin`,`/status/list` | `mcp_processor.rs`,`mcp_refresh.rs` | C→S req | MCP ops | `mcpServer/*` |
| `skills/{list,extraRootsSet,configWrite}` | `config_processor.rs`,`skills_watcher.rs` | C→S req | Skills mgmt | `skills/changed` |
| `hooks/list` | `config_processor.rs` | C→S req | List hooks | `hook/{started,completed}` |
| `plugin/{list,read,install,uninstall}` | `plugin_processor.rs` | C→S req | Plugin mgmt | `app/list/updated` |
| `marketplace/{add,remove,upgrade}` | `marketplace_processor.rs` | C→S req | Marketplace | — |
| `account/{status,login,logout,get,rateLimits}` | `account_processor.rs` | C→S req | Auth/account | `account/{updated,rateLimits/updated}` |
| `collaboration-mode/list`,model/config | `config_processor.rs`,`config_manager.rs` | C→S req | Modes/config | `configWarning` |
| `command/exec` | `command_exec.rs`,`command_exec_processor.rs` | C→S req | Terminal exec | `command/exec/outputDelta` |
| `process/*` | `process_exec_processor.rs` | C→S req | Process spawn (exp) | `process/{outputDelta,exited}` |
| `environment/add` | `environment_processor.rs` | C→S req | Env config | — |
| `git/diffToRemote` | `git_processor.rs` | C→S req | Git diff | — |
| `feedback/upload` | `feedback_processor.rs` | C→S req | Feedback | — |
| `thread/realtime/*` | `turn_processor.rs` | C→S req | Realtime voice (exp) | `thread/realtime/*` |
| Background: skills watcher | `skills_watcher.rs` | job | Watch skill files | `skills/changed` |
| Background: mcp refresh | `mcp_refresh.rs` | job | Poll MCP status | `mcpServer/startupStatus/updated` |
| Background: fs watch | `fs_watch.rs` | job | Native FS events | `fs/changed` |
| Background: daemon update loop | `app-server-daemon/update_loop.rs` | job | Auto-update binary | — |

### 15.3 TUI slash commands (each = a feature row)

`/model`, `/ide`, `/permissions`, `/keymap`, `/vim`, `/setup-default-sandbox`,
`/sandbox-add-read-dir`, `/experimental`, `/approve`, `/memories`, `/skills`, `/hooks`,
`/review`, `/rename`, `/new`, `/archive`, `/resume`, `/fork`, `/init`, `/compact`,
`/plan`, `/goal`, `/agent` (`/subagents`), `/side` (`/btw`), `/copy`, `/raw`, `/diff`,
`/mention`, `/status`, `/debug-config`, `/title`, `/statusline`, `/theme`, `/pets`,
`/ps`, `/stop` (`/clean`), `/clear`, `/personality`, `/realtime`, `/settings`,
`/mcp`, `/plugins`, `/apps`, `/logout`, `/rollout`, `/quit` (`/exit`), `/feedback`.
(Defined in `tui/src/slash_command.rs`; dispatched in `chatwidget/slash_dispatch.rs`.)

### 15.4 Tool handlers (each = a feature row)

`shell`, `unified_exec`, `apply_patch`, `plan`, `goal`, `agent_jobs`, `multi_agents`(v1/v2),
`mcp`, `mcp_resource` list/read, `request_permissions`, `request_user_input`,
`request_plugin_install`, `tool_search`, `view_image`, `extension_tools`, `dynamic`,
`code_mode` execute/wait, `web_search` (ext), `image_generation` (ext), `memories` (ext).

### 15.5 Hooks, providers, stores, services (each = a feature row)

- **Hook events:** `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PreCompact`, `Stop`, `PermissionRequest`.
- **Model providers:** `openai`, `amazon-bedrock`, `ollama`, `lmstudio` (+ user-defined).
- **Auth providers:** Bearer, command-based, AWS SigV4, ChatGPT session.
- **Stores:** rollout (JSONL), state (sqlite), thread-store, message-history, memories, agent-graph-store, keyring-store.
- **Services:** app-server, app-server-daemon, exec-server, mcp-server, responses-api-proxy, network-proxy, stdio-to-uds, realtime-webrtc.

---

## 16. Patterns, Tech Debt, Risks & Improvements

### Patterns
Event-driven streaming (async channels), actor-like submission loop, immutable
`TurnContext`, JSON-RPC request/response correlation with scoped serialization,
event-sourced persistence (rollout), command/sandbox **policy transformation**,
centralized **approval orchestration**, multi-call binary (`arg0`), compile-time
**protocol→TS/JSON schema** generation, capability/feature flags, MITM-proxy network
egress control, Guardian as a sub-agent for auto-approval (CQRS-ish: read-side review
session separate from write-side execution).

### Hidden/implicit features
Subagents/multi-agent (`/agent`), ephemeral side conversations (`/side`,`/btw`),
terminal "pets" (`/pets`), realtime voice (WebRTC, macOS), Guardian auto-approval,
long-term memories with consolidation, code-mode V8 execution PoC, attestation,
remote-control tunneling, external-agent session migration, agent-identity JWT,
dotslash distribution, devcontainer firewall.

### Technical debt
Several **very large files** (`chat_composer.rs` ~11k LOC, `thread_processor.rs` ~4.2k,
`config/mod.rs` ~3.8k, `bespoke_event_handling.rs` ~3.8k) — candidates for decomposition.
Dual exec policy engines (`execpolicy` + `execpolicy-legacy`). Deprecated protocol
variants (`ExecCommandApproval`/`ApplyPatchApproval` vs new `item/*/requestApproval`).
v1↔v2 duplication (`multi_agents`, `compact_remote`/`_v2`). `v8-poc` is exploratory.

### Scalability concerns
Per-thread serialization in app-server can bottleneck heavy multi-client/IDE usage;
SSE/WS idle/reconnect tuning matters under load; sqlite state DB is single-writer.

### Security risks (and mitigations present)
Tool execution is the core risk surface — mitigated by sandboxing (landlock/seatbelt/
bwrap/win-ACL), execpolicy Starlark rules, approval modes + Guardian, network MITM proxy
allowlist, process-hardening (no ptrace/core dumps), secret sanitizer, keyring storage,
0600 auth.json. Residual: `danger-full-access` mode, MITM proxy holds a CA, plaintext
`auth.json` when keyring unavailable, MCP servers run arbitrary processes.

### Enterprise-grade improvements (suggested)
Managed-config/MDM hardening (already partial via `requirements.toml`), centralized
audit log shipping (otel already present), policy-as-code distribution for execpolicy,
RBAC on app-server connections, signed plugin/skill bundles, per-tenant rate limiting,
break up mega-files, unify v1/v2 paths, retire legacy execpolicy.

---

## 17. Critical Files to Understand First

1. `codex-rs/cli/src/main.rs` — entry & subcommand routing.
2. `codex-rs/core/src/session/mod.rs` + `session/handlers.rs` — the agent submission loop.
3. `codex-rs/core/src/tools/router.rs` + `tools/orchestrator.rs` — tool dispatch + approval/sandbox.
4. `codex-rs/core/src/client.rs` + `codex-api` endpoints — model calls/streaming.
5. `codex-rs/app-server/src/message_processor.rs` + `request_processors/turn_processor.rs` — IPC.
6. `codex-rs/app-server-protocol/src/protocol/common.rs` — the entire IPC API surface.
7. `codex-rs/sandboxing/src/manager.rs` + `linux-sandbox`/`windows-sandbox-rs` — security.
8. `codex-rs/tui/src/app.rs` + `app_server_session.rs` + `bottom_pane/chat_composer.rs` — UI.
9. `codex-rs/core/src/config/mod.rs` — configuration model.
10. `sdk/typescript/src/codex.ts` + `sdk/python/src/openai_codex/client.py` — SDK entry.

---

## 18. How To Rebuild This App From Scratch (Roadmap)

1. **Protocol & types first** — define `protocol` (Op/Event/ResponseItem) and the
   `app-server-protocol` (JSON-RPC client/server requests + notifications) with schema
   generation to TS/JSON. This is the contract for every frontend.
2. **Model client** — `model-provider-info` + `codex-api` (Responses API SSE/WS) +
   `models-manager`; support OpenAI first, then local (ollama/lmstudio) and Bedrock.
3. **Core engine** — `Session`/`ThreadManager` + submission loop + `RegularTask`; the
   model↔tool loop; event streaming over async channels.
4. **Tools + sandbox** — `tools` registry, `shell`/`apply_patch` handlers, `ToolOrchestrator`,
   then `execpolicy` (Starlark) and per-platform sandbox (`landlock`/`seatbelt`/win) +
   `network-proxy` + `process-hardening`.
5. **Persistence** — `rollout` JSONL + `state` sqlite index; resume/fork.
6. **App-server** — wrap core in JSON-RPC with transports (stdio/uds/ws), approval
   server-requests, scoped serialization, daemon + auto-update.
7. **Frontends** — TUI (ratatui) via app-server-client; then native desktop/IDE clients
   using the generated protocol types; SDKs (TS spawns `exec`; Python drives app-server).
8. **Auth** — `login`/`chatgpt` OAuth + device code + API key; `auth.json`/keyring; refresh.
9. **Extensibility** — MCP (client + server), plugins + marketplace, skills, hooks,
   connectors, `ext/*` extensions, Guardian, memories.
10. **Observability & ops** — `otel`, `analytics`, `feedback`, doctor; cloud-tasks for
    Codex Web; packaging (`codex-cli` npm, Bazel/Cargo, Nix, code-signing CI).

---

## 19. Codex-App Internal Knowledge Base (one-paragraph summary)

Codex-App is a Rust monorepo implementing a local AI coding agent. The **`core`** crate is
an event-driven engine: a `Session` runs a submission loop that, per user turn, builds an
immutable `TurnContext`, streams from the OpenAI **Responses API** (`codex-api`, SSE + WS),
and routes model tool calls through a `ToolRouter`/`ToolOrchestrator` that enforces
**execpolicy** (Starlark) rules, **approval** modes (with an optional **Guardian**
auto-approval sub-agent), and per-platform **sandboxing** (landlock/seatbelt/bwrap/Windows
ACLs) plus a **MITM network proxy** and **process-hardening**. Tools include shell,
apply_patch, unified_exec (PTY), MCP, plan, multi-agent, web_search, and image_generation.
Sessions are persisted as **rollout JSONL** indexed by a **sqlite state DB**, with
long-term **memories** and cross-session history. The engine is fronted by the
**`app-server`** — a JSON-RPC service (stdio/unix/websocket) whose typed protocol is
code-generated into **TypeScript/JSON Schema** for the native **Desktop App**, **IDE
extensions**, and the **Python SDK**; the **TUI** (ratatui) and **TypeScript SDK** are the
other clients. Authentication is ChatGPT OAuth / device code / API key / agent-identity JWT
stored in `~/.codex/auth.json` (or OS keyring). The whole thing ships as a single
multi-call binary (`arg0` dispatch) distributed via npm (`codex-cli`), Homebrew, Nix, and
GitHub releases, with `cloud-tasks` bridging to the cloud **Codex Web** agent.
