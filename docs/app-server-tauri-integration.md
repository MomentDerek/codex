# Integrating `codex app-server` into a Tauri-based `codex-gui`

This guide shows how to embed the `codex app-server` JSON-RPC process inside a Tauri desktop app (`codex-gui`). It focuses on launching the server binary, wiring stdio to Tauri commands/events, and implementing the core thread/turn/auth flows exposed by the protocol.

## Prerequisites

- Ship the `codex` CLI binary alongside the Tauri bundle or ensure it is on `PATH`. The app-server is invoked as a subcommand: `codex app-server`.
- Use JSON-RPC 2.0 messages without the `"jsonrpc":"2.0"` header, streamed as JSONL over stdio.【F:codex-rs/app-server/README.md†L16-L19】
- Generate strongly typed bindings for your frontend: `codex app-server generate-ts --out <dir>` to emit a TypeScript schema that matches the bundled binary version.【F:codex-rs/app-server/README.md†L20-L27】

## Process lifecycle in Tauri

1. **Spawn the server**
   - Use `tauri::async_runtime::spawn` with `tokio::process::Command` (or `std::process::Command` wrapped in a blocking task) to launch `codex app-server`.
   - Configure `stdin`/`stdout` as piped; ignore `stderr` or surface it to a debug console.
   - Optionally set the working directory to the user’s current project root.
2. **Stream JSONL**
   - Write each JSON-RPC request as a single line ending in `\n`.
   - Read stdout line-by-line and deserialize to route responses vs. notifications; forward notifications to the frontend via `app.emit_all`.
3. **Graceful shutdown**
   - On app exit, send `turn/interrupt` for in-flight turns, then kill the child process if it has not exited.

## Required initialization handshake

`codex app-server` rejects all methods until the client performs the one-time handshake: send `initialize` then emit `initialized`. Include `clientInfo` to identify your GUI.【F:codex-rs/app-server/README.md†L41-L67】 Example request body:

```json
{ "method": "initialize", "id": 0, "params": { "clientInfo": { "name": "codex-gui", "title": "Codex GUI (Tauri)", "version": "0.1.0" } } }
```

After receiving the `initialize` result, immediately send an `initialized` notification. Cache the returned user agent string if you surface it in diagnostics.

## Core conversation flow

### Threads
- Start new conversations with `thread/start`; you receive a `thread` object and a `thread/started` notification.【F:codex-rs/app-server/README.md†L41-L111】
- Resume prior sessions with `thread/resume` using the stored `threadId`; no extra notification is emitted.【F:codex-rs/app-server/README.md†L113-L118】
- Build a history view with `thread/list`, handling `cursor` pagination.【F:codex-rs/app-server/README.md†L120-L145】
- Archive conversations with `thread/archive` to hide them from subsequent lists.【F:codex-rs/app-server/README.md†L146-L155】

### Turns and streaming items
- Call `turn/start` with user input (text or images) plus optional overrides such as `cwd`, `sandboxPolicy`, `model`, and `effort`. The response echoes the turn and the server begins streaming `turn/started`, `item/*`, and `turn/completed` events.【F:codex-rs/app-server/README.md†L157-L189】【F:codex-rs/app-server/README.md†L298-L304】
- Handle incremental content with the invariant `item/started` → zero or more deltas → `item/completed`. Persist items to render the full transcript as updates arrive.【F:codex-rs/app-server/README.md†L298-L304】
- Interrupt a running turn via `turn/interrupt` and wait for the terminal `turn/completed` event with `status: "interrupted"`.【F:codex-rs/app-server/README.md†L191-L204】

### Reviews and one-off commands
- Trigger automated reviews with `review/start`, choosing inline or detached delivery depending on whether the review should stay on the same thread. Watch for `enteredReviewMode` and `exitedReviewMode` items to render progress and the final review text.【F:codex-rs/app-server/README.md†L205-L273】
- Run sandboxed utilities without a thread using `command/exec`; honor the optional `cwd`, `sandboxPolicy`, and `timeoutMs` arguments and display `stdout`/`stderr` from the result.【F:codex-rs/app-server/README.md†L274-L293】

### Approval flows
- **Shell commands**: when Codex proposes a command, the server sends `item/commandExecution/requestApproval`; reply with `accept`/`decline`, then wait for the final `item/completed` carrying execution output.【F:codex-rs/app-server/README.md†L300-L304】
- **File changes**: render the diff chunks from the initial `fileChange` item, answer `item/fileChange/requestApproval`, and update UI from the concluding `item/completed` status.【F:codex-rs/app-server/README.md†L389-L396】

## Authentication UX

Use the auth/account methods to manage credentials and rate limits.【F:codex-rs/app-server/README.md†L398-L497】

- Check current state with `account/read`; `requiresOpenaiAuth` indicates whether the active provider needs OpenAI creds.【F:codex-rs/app-server/README.md†L402-L435】
- API key login: call `account/login/start` with `{type:"apiKey"}` and listen for `account/login/completed` + `account/updated`.【F:codex-rs/app-server/README.md†L436-L455】
- ChatGPT login: start `{type:"chatgpt"}`, open `authUrl` in the system browser (Tauri `shell::open`), and wait for `account/login/completed` + `account/updated`.【F:codex-rs/app-server/README.md†L456-L468】 Provide a cancel action that calls `account/login/cancel`.【F:codex-rs/app-server/README.md†L470-L475】
- Implement logout via `account/logout` and re-read account state as needed. Rate-limit panels can subscribe to `account/rateLimits/updated`.【F:codex-rs/app-server/README.md†L477-L497】

## Wiring Tauri commands and events

- Expose a Tauri command such as `start_app_server` that spawns the process and stores a handle in state. Create a message channel that forwards stdout lines to the frontend as events (e.g., `app.emit_all("codex:rpc", payload)`).
- Add a command `send_rpc` that accepts `{id?, method, params}` from the frontend, serializes to JSON, and writes a line to the server’s stdin. Protect writes with a mutex to avoid interleaving.
- Maintain a map of pending `id` → `oneshot::Sender` so responses can be awaited in Rust and forwarded back to the webview, while notifications are broadcast immediately.

## Frontend data flow (React/Svelte/Vanilla)

- Hydrate the UI from the latest thread snapshot and append incoming item deltas in order.
- Use the generated TypeScript schema to validate payloads and drive discriminated-union rendering for `item` types (agent text, reasoning, tool output, file edits, etc.).
- Implement optimistic UX for approval prompts: show the request immediately, disable duplicate submissions, and surface the terminal `item/completed` status.

## Packaging tips for `codex-gui`

- Bundle the `codex` binary in `resources/` and resolve it at runtime with `tauri::api::path::resource_dir`.
- On macOS and Windows, ensure the child process inherits the app sandbox allowances necessary for filesystem access you expect to permit.
- Keep the app-server version in sync with the generated TypeScript bindings; regenerate on upgrades.

## Debugging and diagnostics

- Mirror server `stderr` to a Tauri window or log file during development to capture protocol errors.
- To validate end-to-end wiring, you can run the `codex-app-server-test-client` binary from the same bundle; it exercises initialization and sample turn calls.
- Include a developer toggle to dump raw JSONL traffic for troubleshooting handshake and approval flows.
