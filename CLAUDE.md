# CLAUDE.md

## Project Overview

`clipshot` is a unified Rust binary for clipboard and file sync across devices.

It has three runtime faces built on the same daemon/core:

1. **Tauri GUI mode** - desktop app with tray icon, dock menu, hotkeys, React UI, and native clipboard access.
2. **Browser UI mode** - same daemon + HTTP API + embedded React app, launched with `--browser` or `settings.use_browser_ui = true`.
3. **CLI/headless mode** - daemon/service plus one-shot commands (`send`, `push`, `pair`, `setup`, `share-uri`, `add-uri`, `history`, `retry-transfer`, `update`, `config`, `doctor`, etc.).

### Auto-detection

`src/main.rs` decides startup mode like this:

- `clipshot` with **no explicit subcommand**:
  - if `--browser` is present -> launch browser UI
  - else if built with `gui` feature and display is available -> launch Tauri GUI
  - else -> CLI error/help path
- `clipshot <subcommand>` -> always CLI mode
- `clipshot --browser` works even without a display

On Linux, display detection uses `DISPLAY` / `WAYLAND_DISPLAY`. On macOS and Windows, GUI is assumed available when built with `gui`.

## Quick Commands

### Build

```bash
# Full desktop build (Tauri + embedded frontend)
cd web && bun install && bun run build && cd ..
cargo build

# Release desktop build
cd web && bun run build && cd ..
cargo build --release

# Tauri GUI installer (DMG on macOS, NSIS on Windows)
cd web && bun run build && cd ..
cargo tauri build

# Headless/server build (no Tauri / no GTK / no WebKit)
cargo build --release --no-default-features --features iroh

# Full release pipeline (all platforms)
./scripts/build-release.sh --all
./scripts/build-release.sh --macos    # DMG + binary
./scripts/build-release.sh --windows  # NSIS installer (Windows VM)
./scripts/build-release.sh --linux    # GUI binary (default features)
```

### Run

```bash
# Auto mode: GUI on desktop, CLI on headless
cargo run

# Browser UI mode
cargo run -- --browser

# Headless daemon
cargo run -- daemon --port 19231

# Headless daemon with browser HTTP API
cargo run -- daemon --port 19231 --http-port 15282

# Pairing (6-digit numeric code)
cargo run -- pair
cargo run -- pair 482917

# Share / join URI
cargo run -- share-uri --port 19231
cargo run -- add-uri 'clipshot://node/...'

# History / retry
cargo run -- history --filter all --limit 20
cargo run -- retry-transfer ~/.clipshot/sync/latest.png
```

### Test / lint

```bash
# Rust
cargo test --lib
cargo clippy --all-targets -- -D warnings

# Frontend
cd web && bun x vitest run

# Browser-mode Playwright
cd web && PLAYWRIGHT_PROJECT=browser-mode bun x playwright test

# Docker API E2E
cd tests/e2e && docker compose -f docker-compose.api.yml up -d
cd ../../web && bun x playwright test --project=docker-api

# Docker relay E2E
cd tests/e2e && docker compose -f docker-compose.relay-e2e.yml up -d
cd ../../web && bun x playwright test --project=docker-relay
```

### Release / deploy

```bash
# Cross-platform release artifacts
./scripts/build-release.sh

# One-line install
curl -fsSL https://clipshot.cc/install.sh | bash
# Then pair:
clipshot pair              # generates 6-digit code
clipshot pair 482917       # joins with code from other device
clipshot setup             # opens browser for account (optional)

# Remote Linux build example
ssh aws-gene "cd ~/clipshot-repo && source ~/.cargo/env && git pull origin main && cargo build --release"
ssh aws-gene "cp ~/clipshot-repo/target/release/clipshot ~/clipshot/clipshot"
```

## Architecture

### Unified Binary

```text
clipshot (single binary)
├─ Desktop / GUI path
│  ├─ Tauri window + tray + dock + hotkeys
│  ├─ React web app embedded into desktop binary
│  └─ DaemonBridge -> AsyncDaemon
├─ Browser UI path
│  ├─ tiny_http REST API + embedded React SPA
│  └─ AsyncDaemon via control/cache layer
└─ CLI / headless path
   ├─ One-shot commands
   ├─ Long-running daemon / service
   └─ AsyncDaemon / App core
```

### Daemon-Centric Design

All business logic is supposed to live in `AsyncDaemon` or center modules under `src/p2p/control/`.

```text
                           ┌─────────────────────┐
                           │    AsyncDaemon      │
                           │  business logic     │
                           └─────────┬───────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
      ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
      │ CLI / control │      │ HTTP handlers │      │ DaemonBridge  │
      │ thin wrapper  │      │ thin wrapper  │      │ Tauri bridge  │
      └───────────────┘      └───────────────┘      └───────────────┘
                                                             │
                                                             ▼
                                                     Tauri commands / UI
```

Rules that the codebase already follows:

- HTTP handlers queue `PeerCommand`s or read caches.
- Tauri commands are thin wrappers around `DaemonBridge` / control functions.
- Shared response types are canonical center structs, not per-interface copies.
- UI-specific behavior belongs in `gui/` only: tray, dock, hotkey orchestration, immediate icon feedback.

### Module Map

Every `src/` directory:

| Directory | Purpose |
|---|---|
| `src/cli/` | Clap command definitions, browser-mode entry, control-socket client paths, service management. |
| `src/clipboard/` | Cross-platform clipboard implementations, restore helpers, platform-specific reading/writing. |
| `src/doctor/` | Self-diagnosis checks for binary/config/clipboard/daemon/peers/hub. |
| `src/gui/` | Tauri app setup, bridge, tray/dock/hotkeys, commands, browser/UI integration helpers. |
| `src/http/` | tiny_http server, REST routes, typed request parsing, SPA/static asset serving. |
| `src/network/` | mDNS discovery, interface enumeration, DNS/address prioritization/filtering. |
| `src/output/` | OSC52/stdout output adapters and output composition. |
| `src/p2p/` | High-level P2P control surface and v2 daemon integration. |
| `src/p2p/control/` | Global caches, typed control commands, activity/transfers/history, discovery, settings bridge. |
| `src/p2p/v2/` | AsyncDaemon, peer registry, hub client, URI processing, native QUIC sync (wire/sync_protocol/sync_handler), connection_monitor, broadcast_queue, gossip. |
| `src/settings/` | TOML-backed settings/peers/activity/transfers stores, validation, migration. |
| `src/sync/` | File sync manager, outbox/catch-up sync, channel abstractions, legacy sync helpers. |
| `src/sync/channel/` | Sync channel implementations for bidirectional sync paths. |
| `src/transport/` | SSH/SCP transport for classic CLI `send` / remote bridge use case. |

Important non-directory root modules from `src/lib.rs`:

- `content` - canonical clipboard payload representation (`text`, `image`, `file`)
- `error` - typed application errors + user-facing suggestions
- `logging` - file/stderr logging setup
- `paths` - config/data/control-socket locations
- `sync_status` - global sync-state singleton
- `validation` - input validation helpers
- `compress` - LZ4 / image compression helpers
- `service` - systemd/launchd integration
- `config` - legacy config helpers
- `crash_report` - crash report collection
- `debug_storage` - debug/diagnostic storage utilities
- `pair_v2` - unified pairing: PairCode, PairOffer, PairBroker, DH handshake, runtime keypair store
- `permissions` - macOS Input Monitoring and permission checks
- `autostart` - OS login-item / autostart integration
- `clipboard_cache` - clipboard content cache for dedup
- `error_reporter` - error reporting utilities
- `tls` / `signal` / `display_utils` / `utils` - support infrastructure

### Web Frontend

Stack and behavior inferred from `web/src/`:

- React + TypeScript
- `HashRouter` routing
- UI shell with sidebar + top status bar
- Data access via hooks around `@/lib/tauri` IPC wrappers
- Automatic **HTTP fallback** via `web/src/lib/api/endpoints.ts` when Tauri IPC is not available
- Toasts via `sonner`
- Polling-heavy dashboard instead of subscriptions

#### Routes

`web/src/App.tsx`:

- First-run mode (no `group_token`):
  - `*` -> `WelcomeScreen`
  - `/settings` -> standalone settings shell
- Normal mode:
  - `/` -> redirect `/overview`
  - `/overview` -> `OverviewPage`
  - `/peers` -> `PeersPage`
  - `/history` -> `HistoryPage`
  - `/activity` -> redirect `/history`
  - `/transfers` -> redirect `/history`
  - `/settings` -> `SettingsPage`

#### Layout / navigation

- `Layout.tsx` shows top header with `SyncToggleButton` (icon-only pause/play/spinner), connected-peer badge, and theme toggle.
- `Sidebar.tsx` provides Overview / Peers / History / Settings, collapsible sidebar state in `localStorage`, and a single **Pair device** entry point.
- `PairDialog.tsx` is the canonical pairing UX used from sidebar and peer-add flows.

#### Pairing UX

`PairDialog` is now a **single-entry** flow with a 2-button menu:

1. **Generate code** - creates one 6-digit code (`482917`), shows spinner "Waiting for other device...", then shows 4 confirmation digits after joiner arrives (mutual-auth DH)
2. **Enter code** - accepts only digits, resolves via PairBroker, completes X25519 DH, then shows 4 confirmation digits derived from the shared secret

No tabbed mode selection. All legacy pair code (`src/pair.rs`, `handlers_local_pair.rs`, Tauri commands `pair_generate_code`/`pair_join_code`/`pair_generate_local`/`pair_join_local`) has been deleted.

#### Hook polling intervals

- `useDaemonStatus` - every **2s**, plus immediate refetch on `sync-state-changed`
- `usePeers` - every **2s**, with 3s disconnect grace to reduce UI flapping
- `useActivity` - every **5s**
- `useTransfers` - every **500ms**
- `useHistory` - adaptive: **500ms** when any transfer is `in_progress`, else **5s**
- `useSettings` - every **5s** so `is_pro` / `central_relay_enabled` changes from hub cache are visible
- `usePinnedPeers` - localStorage only, no backend
- `useClipboardToggleHotkey` - listens for `hotkey-toggle-clipboard` and calls `toggle_clipboard_mode()`

#### Browser HTTP fallback limits

`web/src/lib/api/endpoints.ts` does not perfectly mirror native Tauri behavior:

- `toggle_clipboard_mode` in browser mode degrades to `POST /api/sync/toggle`
- `restart_app`, `open_containing_folder`, `copy_file_path` are unavailable in browser mode
- `discover_peers` returns `[]` in browser fallback
- direct clipboard writes use `navigator.clipboard` when possible

## Key Architecture Decisions

1. **Single binary, multiple faces**
   One binary serves desktop GUI, browser UI, and CLI/headless workflows.

2. **AsyncDaemon is the center**
   Business logic belongs in `AsyncDaemon` and center modules, not in handlers or UI wrappers.

3. **Canonical shared types**
   `DaemonStatusFull`, `FullPeerInfo`, `Settings`, `ActivityEvent`, `TransferInfo`, `HistoryEntry`, `ShareUriInfo`, `DiscoveredPeer` are the shared interface types used by GUI, HTTP, and web bindings.

4. **Single global sync state**
   `src/sync_status.rs` owns a global `OnceLock<SyncStatus>` backed by atomics. Tray icon, HTTP API, browser UI, and Tauri UI all read the same source.

5. **Sync state priority is centralized**
   `compute_sync_state(paused, connected, sent, received)` currently yields: `Paused > Sending > Receiving > NoPeers > Idle`.

6. **Immediate UI feedback on pause/resume**
   `DaemonBridge::pause_sync()` and `resume_sync()` update the global sync state and mirror atomic immediately before daemon-side work, so tray/UI reacts instantly.

7. **Hotkey/tray/dock toggles are serialized**
   `tauri_app.rs` routes `hotkey-toggle-sync` events through a `HotkeyToggleWorker` queue to avoid race conditions between native hotkey, tray menu, and dock menu.

8. **Clipboard and network stages are separated per tick**  
   Tick stages: Clipboard → FastOps → IncomingAndGossip → BroadcastPending → SlowReconnect. Clipboard stage detects changes + saves locally + queues to BroadcastQueue (never blocks on network). BroadcastPending stage drains queue and sends. GUI mode runs clipboard/network in separate lock scopes with panic recovery.

9. **Outbox-based catch-up sync**
   `src/sync/outbox.rs` keeps an append-only, GC'd outbox (100 entries, 24h TTL) with per-peer delivery tracking and optional inline data bytes (≤10MB). Seeded from `~/.clipshot/sync/` on startup. Active broadcast uses `BroadcastQueue` (bounded FIFO); outbox is for persistence/catch-up only.

10. **mDNS is feature-gated**
    Local discovery exists only with `mdns-discovery`; otherwise discovery is a clean no-op returning an empty list.

11. **Iroh identity is persistent**
    `peer_connect.rs` persists `~/.config/clipshot/iroh_secret` so the `iroh://<endpoint>` identity survives restarts.

12. **HTTP API is unauthenticated by design, so it is localhost-only by default**
    `CLIPSHOT_HTTP_BIND` can override bind address for CI/docker, but `0.0.0.0` is explicitly warned as unsafe.

13. **CORS defaults are narrow**
    By default only `localhost`, `127.0.0.1`, `https://localhost`, and `tauri://` origins are allowed. Custom origins replace, not extend, defaults.

14. **Pro state is hub-authoritative**
    `is_pro` and `central_relay_enabled` must come from hub subscription updates, not from HTTP POST `/api/settings`.

15. **Browser/Tauri share the same app model**
    The React app prefers Tauri IPC but can fall back to HTTP; both should expose the same semantics wherever possible.

16. **Tray and Dock menu use static `Toggle Sync` label (not dynamic Pause/Resume)**
    Dynamic menu label changes caused ghost clicks on macOS; both the tray and Dock menus use a static `Toggle Sync` item and route through the unified toggle worker.

17. **Dock/taskbar progress is driven from transfer cache**
    `TransferProgressTracker` reads global transfers, updates progress bar, and on macOS prevents sleep with `caffeinate` while transfers are active.

18. **TCP listener is effectively disabled in current daemon startup**
    `AsyncDaemon::start_listener()` is a no-op that only stores `listen_port`; current peer connect logic is `iroh`-first / `iroh`-only in `connect_to_addr()`.

19. **Peer registry canonicalizes iroh keys**
    Bare/canonical `iroh://<hex>` is used as registry key; richer forms with relay/addrs live in peer address lists for probing/reconnect.

20. **Settings roundtrip must include all fields**
    Tauri `save_settings(settings)` expects the full struct; partial frontend bindings can silently erase values because serde fills missing fields with defaults.

21. **Windows clipboard write via hidden HWND**
    `EmptyClipboard()` requires window ownership. A persistent `HWND_MESSAGE` window is created on a dedicated thread with a message pump (`src/clipboard/windows.rs`). `OpenClipboard(hwnd)` + `SetClipboardData` works from any thread/process context. Uses `windows-sys` crate for raw Win32 API.

22. **paste_path_hotkey listener in both GUI and daemon mode**
    `paste_path_hotkey` (Cmd+B / Ctrl+B) is registered in both `tauri_app.rs` (GUI) and `daemon_entry.rs` (daemon mode). On Windows, daemon mode with `--force` flag is the primary deployment mode.

23. **Tauri GUI is the primary Windows distribution**
    Windows builds include Tauri GUI with WebView2, NSIS installer, tray icon, and all hotkeys. Daemon-only mode is secondary. NSIS installer produced by `cargo tauri build`.

24. **DMG installer for macOS**
    `cargo tauri build` produces `.app` bundle. DMG is created manually with `hdiutil` including an `Applications` symlink for drag-to-install. `xattr -cr` clears quarantine for ad-hoc signed binaries.

25. **Non-blocking clipboard broadcast**
    `tick_stage_clipboard` never awaits network I/O. Clipboard changes are saved locally + pushed to `BroadcastQueue`. Network stage (`BroadcastPending`) drains the queue and sends. This prevents clipboard polling stalls during slow broadcasts.

26. **BroadcastQueue is separate from Outbox (SRP)**
    `BroadcastQueue` (bounded FIFO, configurable 1-10) holds data bytes for active send. `Outbox` (100 entries, 24h TTL) holds metadata for catch-up on reconnect. Clipboard stage writes to both.

27. **Consistent filenames across nodes**
    Sender generates `img_YYYYMMDD_HASH.ext` and includes it in wire header `name` field. Receiver uses sender's filename directly (not re-generating locally). Hash is computed from data bytes only (no timestamp) so identical data → identical name on all nodes.

28. **Post-quantum key exchange is opt-in**
    `cargo build --features pq` enables ML-KEM + X25519 hybrid key exchange via aws-lc-rs. Default build uses ring (lighter, better cross-compile support).

29. **Pairing is a single primitive in v2 with mutual-auth DH**
    Pair onboarding: one numeric 6-digit code + X25519 ephemeral key exchange + 4 confirmation digits derived from DH shared secret. Transport race resolution via `PairBroker` (mDNS and Portal `/api/pair/v2`). Flow: Device A generates code → Device B joins (sends eph_pub via portal) → Device A polls for joiner → both compute `DH(priv, other_pub)` → identical 4-digit confirmation. MITM produces divergent digits. Reciprocal peer add: both devices add each other automatically. `group_token` ensured at pair time (account-decoupled onboarding).

## HTTP API Reference

### Base behavior

- Default bind: `127.0.0.1:<port>`
- Override bind: `CLIPSHOT_HTTP_BIND=0.0.0.0|127.0.0.1|::1|localhost`
- JSON envelope for JSON endpoints:

```json
{"success": true|false, "data": ..., "error": null|"message"}
```

- Unknown GET routes fall back to the embedded SPA `index.html`
- `OPTIONS` gets `204` with CORS headers

### Canonical response types

#### `DaemonStatusFull`

Fields returned by `GET /api/status` and Tauri `get_daemon_status`:

- `running: bool`
- `sync_enabled: bool`
- `node_id: string`
- `listen_addr: string`
- `connected_peers: number`
- `total_peers: number`
- `node_name: string`
- `uptime_secs: number`
- `share_uri_expires_secs: number`
- `iroh_enabled: bool`
- `iroh_addr: string | null`
- `messages_sent: number`
- `messages_received: number`
- `hub_connected: bool`
- `hub_url: string | null`
- `hub_limit_reason: string | null`
- `hub_limit_current: number | null`
- `hub_limit_max: number | null`
- `hub_limit_manage_url: string | null`
- `hub_limit_message: string | null`
- `sync_state: "idle" | "sending" | "receiving" | "no_peers" | "paused"`
- `app_version: string`
- `tick_age_ms: number`
- `peer_relay_url: string | null`

#### `FullPeerInfo`

Fields returned by `GET /api/peers` and Tauri `get_peers`:

- `name`
- `address`
- `connected`
- `status: Online | Offline | Connecting`
- `node_id: string | null`
- `addresses: string[]`
- `latency_ms: number | null`
- `last_sync: number | null`
- `transport_type: string | null`
- `limited: bool`
- `reject_reason: string | null`
- `device_type: string | null`
- `device_id: string | null`
- `transfer_bytes_done: number | null`
- `transfer_bytes_total: number | null`
- `transfer_speed_bps: number | null`
- `connection_type: string | null`
- `path_rtt_ms: number | null`

#### `ActivityEvent`

- `id`
- `event_type`
- `description`
- `timestamp`
- `peer_name`
- `content_type?`
- `size_bytes?`
- `error_message?`
- `file_path?`

#### `TransferInfo`

- `id`
- `direction: "sending" | "receiving"`
- `peer_name`
- `peer_addr`
- `content_type`
- `total_bytes`
- `transferred_bytes`
- `status: "in_progress" | "complete" | "failed"`
- `started_at`
- `completed_at?`
- `file_path?`
- `error_message?`
- `chunks_sent`
- `chunks_total`
- `speed_bytes_per_sec`

#### `HistoryEntry`

- `id`
- `entry_type`
- `name`
- `size_bytes`
- `peer_name`
- `direction`
- `status`
- `timestamp`
- `file_path?`
- `content_type`
- `transferred_bytes?`
- `total_bytes?`
- `chunks_sent?`
- `chunks_total?`
- `speed_bytes_per_sec?`
- `error_message?`
- `completed_at?`

### Core / SPA / static routes

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/` | - | embedded `index.html` |
| GET | `/index.html` | - | embedded `index.html` |
| GET | `/assets/*` | - | embedded static asset or `404 text/plain` |
| GET | `/health` | - | `{"status":"ok"}` |
| GET | `/api/icon.png` | - | PNG chosen from current global sync state |
| GET | `/api/permissions` | - | `ApiResponse<PermissionInfo[]>` (macOS Input Monitoring etc.) |
| OPTIONS | any API path | - | `204` + CORS headers |

### Status / activity / share routes

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/api/status` | - | `ApiResponse<DaemonStatusFull>` |
| GET | `/api/activity` | - | `ApiResponse<ActivityEvent[]>` |
| GET | `/api/share_uri` | - | `ApiResponse<{ uri, node_id, expires_in_secs }>` or error if no routable addresses |
| GET | `/api/gossip_peers` | - | same payload as `/api/peers` |
| GET | `/api/transfers` | - | `ApiResponse<TransferInfo[]>` |
| GET | `/api/history?filter=<opt>&limit=<opt>` | - | `ApiResponse<HistoryEntry[]>`, default `limit=100` |
| GET | `/api/history/:id/content` | - | raw file bytes; `image/*` if history entry is image, else `text/plain; charset=utf-8` |
| GET | `/api/last_sync_path` | - | last sync file path |

### Peer routes

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/api/peers` | - | `ApiResponse<FullPeerInfo[]>` |
| POST | `/api/peers` | `{"name":"...","address":"..."}` | queues `PeerCommand::Add` |
| PUT | `/api/peers` | `{"address":"...","name"?:string,"addresses"?:string[],"auth_code"?:string|null}` | queues `PeerCommand::Update` |
| DELETE | `/api/peers/:name` | - | queues remove by peer name or stable hex node ID |
| POST | `/api/peers/:name/connect` | - | queues `ConnectByName` |
| POST | `/api/peers/:name/activate` | - | queues `ActivateByName` |
| POST | `/api/peers/:name/deactivate` | - | queues `DeactivateByName` |
| POST | `/api/peers/:name/reconnect` | - | queues `Reconnect` |
| POST | `/api/add_peer_from_uri` | `{"uri":"clipshot://..."}` | queues `AddFromUri` |

Validation details:

- `AddPeerRequest.name`: 1..64 chars
- `AddPeerRequest.address`: 7..255 chars
- `UpdatePeerRequest.address`: 1..255 chars
- `AddPeerFromUriRequest.uri`: 1..2048 chars, plus non-blank after trim

### Settings routes

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/api/settings` | - | `ApiResponse<Settings>` from daemon-authoritative projection |
| POST | `/api/settings` | partial settings update body | queues `UpdateSettings` |

Accepted update fields in `POST /api/settings`:

Note: `sync_interval_ms` maps to `poll_interval_ms` in the `Settings` struct.

- `auto_sync` (`sync_enabled` alias also accepted)
- `auto_start`
- `notifications`
- `sync_interval_ms`
- `max_file_size_mb`
- `transfer_timeout_per_mb_ms`
- `transfer_timeout_base_ms`
- `chunk_timeout_secs`
- `listen_port`
- `enable_iroh`
- `max_peers`
- `auto_discover`
- `use_browser_ui`
- `direct_send_threshold_mb`
- `hotkey`
- `hub_url`
- `group_token`
- `relay_url`
- `max_peers_free`
- `peer_relay_enabled`
- `n0_relay_enabled`

Not accepted via HTTP API:

- `is_pro`
- `node_password`
- `central_relay_enabled` (hub-authoritative)
- `sync_max_files`
- `sync_max_age_days`
- `sync_max_size_mb`
- `clipboard_hotkey`
- `paste_path_hotkey`
- `cors_origins`
- `theme`
- `poll_interval_ms` (use `sync_interval_ms` API alias instead)
- `retry_attempts`
- `timeout_ms`
- `catchup_limit`
- `broadcast_queue_size`

Validation ranges enforced by `UpdateSettingsRequest`:

- `sync_interval_ms`: 100..3_600_000
- `max_file_size_mb`: 1..10240
- `transfer_timeout_per_mb_ms`: 100..60_000
- `transfer_timeout_base_ms`: 1000..300_000
- `chunk_timeout_secs`: 1..600
- `listen_port`: 1024..65535
- `max_peers`: 1..1000
- `direct_send_threshold_mb`: 1..100
- `max_peers_free`: 1..1000

### Sync / transfer / clipboard routes

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/api/retry_transfer` | `{"file_path":"..."}` | queues retry by path |
| POST | `/api/copy_to_clipboard` | `{"text":"..."}` | queues local clipboard write + broadcast |
| POST | `/api/sync/pause` | - | queues `PauseSync` |
| POST | `/api/sync/resume` | - | queues `ResumeSync` |
| POST | `/api/sync/toggle` | - | queues `ToggleSync` |
| POST | `/api/paste_path` | - | saves clipboard to sync dir, writes path, simulates paste |

Validation:

- `RetryTransferRequest.file_path`: 1..4096 chars
- `CopyToClipboardRequest.text`: 1..1_000_000 chars

### Unified pair routes (`src/http/handlers_pair_v2.rs`)

Single pair API surface for UI and browser fallback.

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/api/pair/generate` | - | `{code, digits: null, expires_in_secs}` |
| POST | `/api/pair/join` | `{code}` | `{digits, peer_addr, joiner_eph_pub_hex}` |
| POST | `/api/pair/poll` | `{code}` | `{ready, joiner_eph_pub_hex, joiner_iroh_addr}` |
| POST | `/api/pair/confirm` | `{code, joiner_eph_pub_hex, joiner_iroh_addr?}` | `{digits}` |
| POST | `/api/pair/abort` | `{code}` | `{aborted, code}` |

Notes:
- `code` is numeric 6-digit (`^[0-9]{6}$`)
- `generate` returns `digits: null` — digits are computed only after joiner arrives
- `join` performs X25519 DH with generator's eph_pub, pushes joiner eph_pub to portal, adds generator as peer
- `poll` is called by generator to check if joiner has arrived (portal `/pair/v2/{code}/confirm`)
- `confirm` computes DH with joiner's eph_pub, adds joiner as peer (reciprocal), returns matching digits
- `abort` cleans the local keypair store; subsequent `confirm` returns 404
- Broker publishes/queries through available discovery transports (`mDNS`, `Portal /api/pair/v2`)

### Test-only daemon HTTP routes (`src/http/handlers_test.rs`)

These require `CLIPSHOT_TEST_HOOKS=1` or return `403`.

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/api/test/clipboard` | - | clipboard snapshot: text returns `type/data/size`, binary returns `type/size` |
| POST | `/api/test/reset` | optional `{"restore_peer_name"?,"restore_peer_addr"?,"max_peers_free"?}` | queues reset-state command |
| POST | `/api/test/kill_iroh_listener` | - | kills the active iroh listener (forces iroh restart on next tick) |

### Tauri hotkey test-hook HTTP server

This is a **separate** tiny_http server started from `src/gui/tauri_app.rs` only when `CLIPSHOT_TEST_HOOKS=1`.

Defaults:

- bind: `127.0.0.1:${CLIPSHOT_TEST_HOOKS_PORT:-18181}`

Routes:

| Method | Path | Body | Response |
|---|---|---|---|
| GET / POST | `/api/test/health` | - | daemon readiness (`200 ready` or `503 starting`) |
| GET / POST | `/api/test/sync_enabled` | - | current `sync_enabled` bool |
| GET / POST | `/api/test/hotkey_metrics` | - | queue/drain latency metrics |
| GET | `/api/test/native_listener_health` | - | native hotkey listener health |
| GET | `/api/test/icon_state` | - | `{ state, state_raw }` from global sync atomic |
| GET | `/api/test/native_key_events` | - | captured native key event snapshot |
| POST | `/api/test/native_key_events` | - | clears captured native key events |
| POST | `/api/test/hotkey_toggle` | optional `X-Clipshot-Source` header | emits `hotkey-toggle-sync` into app event loop |

## Tauri Commands Reference

Current invoke handler count: **37 registered commands** (38 `#[tauri::command]` definitions; `toggle_clipboard_mode` is defined but not registered in invoke_handler - it is called internally from Rust, not from frontend IPC).

### Status / peers

| Command | Params | Returns | Notes |
|---|---|---|---|
| `get_daemon_status` | none | `DaemonStatusFull` | canonical daemon status |
| `get_peers` | none | `FullPeerInfo[]` | full merged peer list |
| `add_peer` | `name: string, address: string, password?: string` | `()` | delegates to bridge add peer |
| `remove_peer` | `peer_id: string` | `()` | remove by address/name/node ID |
| `reconnect_peer` | `name: string` | `()` | reset failures + reconnect |
| `update_peer` | `address: string, name?: string, addresses?: string[], auth_code?: string|null` | `()` | update registry entry |
| `connect_to_peer` | `peer_id: string` | `boolean` | manual connect attempt |
| `activate_peer` | `address: string` | `()` | free-tier active-swap path |
| `deactivate_peer` | `address: string` | `()` | disconnect but keep peer |

### Discovery / share

| Command | Params | Returns | Notes |
|---|---|---|---|
| `get_network_interfaces` | none | `NetworkInterface[]` | enumerates local addresses for current port |
| `export_node_url` | `node_name: string, node_id: string, interfaces: NetworkInterface[]` | `string` | builds signed `clipshot://` URI |
| `add_peer_from_uri` | `uri: string` | `()` | `clipshot://` add path |
| `discover_peers` | none | `DiscoveredPeer[]` | mDNS if feature enabled |

### Activity / settings / pairing

| Command | Params | Returns | Notes |
|---|---|---|---|
| `get_activity` | `limit: number` | `ActivityEvent[]` | reads global activity cache |
| `get_settings` | none | `Settings` | daemon-authoritative effective settings |
| `save_settings` | `settings: Settings` | `()` | persists settings and updates hotkey listener |
| `pair_generate` | none | `{code, digits: null, expires_in_secs}` | generates 6-digit code, stores ephemeral keypair |
| `pair_join` | `code: string` | `{digits, peer_addr, joiner_eph_pub_hex}` | resolves offer via broker, completes DH, adds generator as peer |
| `pair_poll` | `code: string` | `{ready, joiner_eph_pub_hex, joiner_iroh_addr}` | polls portal for joiner arrival |
| `pair_confirm` | `code: string, joinerEphPubHex: string, joiner_iroh_addr?: string` | `{digits}` | computes DH digits, adds joiner as peer (reciprocal) |
| `pair_abort` | `code: string` | `{aborted, code}` | cleans keypair, subsequent confirm returns 404 |

### Sync / transfers / history

| Command | Params | Returns | Notes |
|---|---|---|---|
| `toggle_clipboard_mode` | none | `"image" \| "path"` | path <-> content toggle; **NOT registered in invoke_handler** - called internally from Rust hotkey path, not available via frontend IPC |
| `paste_file_path` | none | `()` | saves clipboard to sync dir, writes path to clipboard, simulates Cmd+V/Ctrl+V paste |
| `pause_sync` | none | `()` | emits `sync-state-changed=false` |
| `resume_sync` | none | `()` | emits `sync-state-changed=true` |
| `get_transfers` | none | `TransferInfo[]` | fast-polled by UI |
| `get_history` | `filter?: string, limit: number` | `HistoryEntry[]` | unified history view |
| `retry_transfer` | `file_path: string` | `string` | bridge retries transfer from disk |
| `copy_to_clipboard` | `text: string` | `()` | local copy + broadcast |
| `copy_file_path` | `path: string` | `()` | local clipboard only |

### OS / shell integration

| Command | Params | Returns | Notes |
|---|---|---|---|
| `open_containing_folder` | `path: string` | `()` | opens parent dir with OS shell |
| `open_external_url` | `url: string` | `()` | only `http://` and `https://` allowed |
| `restart_app` | none | `()` | Tauri process restart |

### Update

| Command | Params | Returns | Notes |
|---|---|---|---|
| `check_for_update` | none | `Option<string>` | checks for available update via Tauri updater or HTTP fallback |
| `install_update` | none | `()` | downloads and installs update |

### Permissions (macOS)

| Command | Params | Returns | Notes |
|---|---|---|---|
| `check_permissions` | none | `PermissionInfo[]` | checks Input Monitoring and other required permissions |
| `open_permission_settings` | none | `()` | opens System Settings for Input Monitoring |
| `restart_app_for_permissions` | none | `()` | restarts app after permission changes |

### Important command semantics

- `toggle_clipboard_mode()`:
  - if clipboard holds a `~/.clipshot/sync/...` text path -> restore original content into clipboard
  - if clipboard holds image/file data -> put latest sync path text into clipboard
  - works for files, not just images
- `save_settings()` updates native hotkey listener immediately via `HotkeyListener::set_hotkey()`
- `pair_generate` ensures a local `group_token` exists (account-decoupled onboarding) before publishing the offer; returns `digits: null` (unknown until joiner arrives)
- `pair_join` performs real X25519 DH, pushes joiner's eph_pub + iroh_addr to portal, adds generator as peer
- `pair_confirm` computes DH with joiner's eph_pub → matching digits; adds joiner as peer (reciprocal)
- `clipboard_hotkey` has `None` default in Settings struct; not exposed in Settings UI, configurable only via `settings.toml`

## Discovery & Connectivity

### Connectivity matrix

| Method | Where it works | Requires | What it does | Notes |
|---|---|---|---|---|
| Hub / Portal WebSocket | Internet / LAN | `hub_url` + `group_token` | node discovery, subscription updates, pairing support | thin control plane only |
| Pair v2 Code | Internet / LAN | running daemon + iroh; optional Portal | 6-digit code + X25519 mutual-auth DH + reciprocal peer add | both devices add each other automatically |
| `clipshot://` Share Link | Any routable path | URI exchange | signed node/share URI with name + addresses + TTL | verified before add |
| `iroh://` direct link | Any path iroh can reach | iroh enabled | direct iroh endpoint import | accepted in PairDialog advanced mode |
| mDNS Local Network | Same multicast LAN | `mdns-discovery` feature + `auto_discover=true` | `_clipshot._tcp.local.` discovery, auto-scan every 60s | auto-adds peers up to free tier limit |
| Manual Entry | Anywhere | known address | add peer by name/address/password | useful for static infra |

### Hub / Portal

Daemon status exposes:

- `hub_connected`
- `hub_url`
- `hub_limit_reason`
- `hub_limit_current`
- `hub_limit_max`
- `hub_limit_manage_url`
- `hub_limit_message`

Pair v2 uses transport race resolution with mutual-auth DH:

- generator publishes `PairOffer` (`code`, `eph_pub`, `iroh_addr`, TTL) via broker
- joiner resolves `code` through available transports (`mDNS`, Portal `/api/pair/v2`)
- joiner performs X25519 DH with generator's eph_pub, pushes own eph_pub + iroh_addr to portal
- generator polls portal for joiner arrival, then computes DH with joiner's eph_pub
- both sides derive identical 4-digit confirmation from the DH shared secret (MITM → divergent digits)
- both sides add each other as peers automatically (reciprocal add via `pair_join` + `pair_confirm`)
- local `group_token` is ensured at pair time (decoupled from account login)

### mDNS

`src/network/discovery.rs`:

- service type: `_clipshot._tcp.local.`
- TXT property: `node_id`
- discovered peer payload: `{ name, address, node_id }`
- registration uses hostname + advertised port
- registration lifetime is tied to `MdnsRegistration` handle; drop unregisters + shuts down daemon

### Iroh listener and addressing

`src/p2p/v2/daemon/peer_connect.rs`:

- `start_listener(port)` does **not** start TCP listener anymore; it only stores `listen_port` for self-address detection.
- `start_iroh_listener(relay_url)` binds an iroh endpoint, persists/stabilizes identity, and builds `iroh://<endpoint>?relay=...&addrs=...` when possible.
- relay mode:
  - custom relay if valid `relay_url`
  - otherwise disabled
- address discovery is timeout-capped:
  - bind timeout: 10s
  - `endpoint.addr()` timeout: 3s
- handshake failure logging is throttled per normalized iroh addr
- bare `iroh://<hex>` can be enriched from registry entries with full relay/addrs form

## Security Model

### HTTP API

- default bind: `127.0.0.1`
- no authentication on HTTP API
- binding to `0.0.0.0` is possible only via `CLIPSHOT_HTTP_BIND` and logs a warning
- CORS default allowlist:
  - `http://localhost*`
  - `http://127.0.0.1*`
  - `https://localhost*`
  - `tauri://*`
- custom CORS origins replace defaults and must match exactly

### Share URI signatures

`clipshot://` URIs are time-limited and signed:

- algorithm: HMAC-SHA256
- secret: stored per node in `~/.config/clipshot/hmac_secret`
- signature: first 8 bytes of HMAC output
- payload includes node ID, expiry, and addresses
- default TTL: 30 minutes (`share-uri --ttl` can override in CLI)

### Iroh identity

- endpoint secret persisted at `~/.config/clipshot/iroh_secret`
- prevents node identity churn across restarts

### Hub auth and subscription

- group token is sent to portal for hub connection / pairing
- `group_token` lives in settings and should not be exposed casually in logs/docs
- `is_pro`
- `node_password` and `central_relay_enabled` are daemon/hub-authoritative and must not be user-set through HTTP API

### Free vs Pro

- `settings.is_pro = false` -> Lite / Free behavior
- free tier active limit controlled by `max_peers_free`
- limited peers remain known in registry but may be grey/inactive in UI
- upgrade path is surfaced in tray menu for non-Pro users

## Sync State Machine

`src/sync_status.rs` defines the global state machine.

### Storage model

- `static GLOBAL: OnceLock<SyncStatus>`
- `SyncStatus.state: Arc<AtomicU8>`
- `SyncStatus.version: Arc<AtomicU64>` for stale auto-reset protection
- `SyncStatus.set_at_ms: Arc<AtomicU64>` for elapsed-since-last-set timing

### States

| State | Value | Meaning |
|---|---:|---|
| `Idle` | 0 | no active transfer |
| `Sending` | 1 | sending to peer(s) |
| `Receiving` | 2 | receiving from peer |
| `NoPeers` | 3 | sync enabled but zero connected peers |
| `Paused` | 4 | user paused sync |

### Writers

- `DaemonBridge::pause_sync()` -> immediate `Paused`
- `DaemonBridge::resume_sync()` -> immediate `Idle`
- `gui/daemon_bridge_state::update_icon_state()` -> post-tick state projection
- `p2p::control::update_sync_state()` -> centralized cache writer path

### Readers

- tray animator
- dock icon state
- HTTP `GET /api/status`
- HTTP `GET /api/icon.png`
- Tauri `get_daemon_status`
- browser/Tauri header `SyncToggleButton`
- hotkey test-hook `/api/test/icon_state`

### Transition logic

`compute_sync_state(paused, connected_peers, sent_this_tick, received_this_tick)`:

1. if paused -> `Paused`
2. else if sent > 0 -> `Sending`
3. else if received > 0 -> `Receiving`
4. else if connected_peers == 0 -> `NoPeers`
5. else -> `Idle`

### Auto-reset behavior

- `Sending` / `Receiving` are set with auto-reset in `update_icon_state()`
- reset duration: 1 second
- version counter prevents old reset tasks from clobbering newer state

## Cache Consistency Model

The project intentionally uses eventual consistency between daemon internals and readers.

### Single-writer caches

Center/global caches under `src/p2p/control/`:

- peer list
- daemon stats
- share URI
- activity events
- transfers
- Pro/relay subscription bits
- sync state projection (via `sync_status::global()`)

### Reader model

Readers include:

- HTTP handlers
- browser UI polling hooks
- Tauri commands
- tray/dock helpers
- CLI/status output

### Guarantees

- daemon is the intended authoritative writer
- readers may observe slightly stale values between ticks/polls
- this is acceptable because clipboard sync UX does not need strict serializable consistency
- lock-heavy code paths are kept out of UI polling loops when possible

## Testing

### Current suite snapshot

Counts requested for this repo snapshot:

| Suite | Count | Command | Typical time | Purpose |
|---|---:|---|---|---|
| Rust lib tests | 1637 | `cargo test --lib` | medium | unit/integration coverage for core modules |
| Frontend Vitest | 271 | `cd web && bun x vitest run` | fast | React/hooks/components/api client |
| Docker API E2E | 43 | `cd tests/e2e && docker compose -f docker-compose.api.yml up -d && cd ../../web && bun x playwright test --project=docker-api` | ~minutes | browser UI against 3-node docker mesh |
| Docker relay E2E | 3 | `cd tests/e2e && docker compose -f docker-compose.relay-e2e.yml up -d && cd ../../web && bun x playwright test --project=docker-relay` | short | relay + blocked direct path |
| Stress mesh | 500 | `bash tests/e2e/scripts/overnight-pair-v2-stress.sh` | ~5h | pair/sync/broadcast/restart across 3-node mesh |
| Clipboard fundamental | 7 | `bash /scripts/clipboard-toggle-test.sh ...` | short | path <-> content toggle correctness |
| Install E2E | 7 | `cd tests/e2e && docker compose -f docker-compose.install.yml up --abort-on-container-exit` | short | installer + pair flow |
| Browser-mode Playwright | project-specific | `cd web && PLAYWRIGHT_PROJECT=browser-mode bun x playwright test` | medium | real local browser-mode daemon |
| VM parallel Playwright | project-specific | `cd web && bun x playwright test --project=vm-parallel` | medium | Tart VM GUI/icon/hotkey coverage |

### Playwright projects (`web/playwright.config.ts`)

| Project | Base URL | Purpose |
|---|---|---|
| `browser-mode` | `http://localhost:15282` | real local `clipshot --browser` |
| `docker-api` | `${DOCKER_NODE1_API:-http://127.0.0.1:19341}` | docker mesh API/browser E2E |
| `docker-relay` | `${DOCKER_NODE1_API:-http://127.0.0.1:8300}` | docker relay E2E |
| `vm-parallel` | `${VM_NODE1_URL:-http://192.168.64.28:15282}` | Tart macOS VM browser-mode/UI E2E |

### Fundamental tests

If these fail, core UX is broken:

- clipboard toggle test (`scripts/clipboard-toggle-test.sh`)
- stress mesh test (`scripts/stress-test.sh`)
- docker-api project for peer management / sync flows
- docker-relay for blocked-direct / relay fallback scenarios

### What to run when

| Changed area | Minimum verification |
|---|---|
| `src/gui/` | `cargo test --lib && cargo clippy --all-targets -- -D warnings` |
| `src/http/` | `cargo test --lib` + docker-api |
| `src/p2p/control/`, `src/p2p/v2/daemon/`, `src/sync/` | `cargo test --lib` + clipboard toggle + stress |
| `src/p2p/v2/hub/` | `cargo test --lib` + docker-api + docker-relay + stress |
| `src/network/` | `cargo test --lib` |
| `web/src/` | `cd web && bun x vitest run` + relevant Playwright project |
| release/build scripts | relevant docker/install/build scripts |

## Build & Deploy

### Cargo feature combinations

```toml
[features]
default = ["gui", "mdns-discovery", "iroh", "relay-server"]
gui = ["dep:tauri", "dep:tauri-build", "dep:rdev", "dep:tauri-plugin-updater", "dep:x11", "dep:enigo"]
mdns-discovery = ["dep:mdns-sd"]
iroh = ["dep:iroh"]
relay-server = ["dep:iroh-relay"]
pq = ["iroh/tls-aws-lc-rs"]  # Post-quantum key exchange (ML-KEM + X25519)
```

Common modes:

- `cargo build` -> desktop build with Tauri + mDNS + iroh
- `cargo build --release --no-default-features --features iroh` -> headless server build
- `cargo run -- --browser` -> browser UI mode with embedded SPA and HTTP API

### Browser mode

Useful when tray/webview dependencies are problematic.

```bash
cd web && bun run build && cd ..
cargo run -- --browser
```

Default health URL in browser mode tests:

```bash
http://localhost:15282/health
```

### Docker

```bash
# Headless image
docker build -f Dockerfile.headless -t clipshot-headless .

# E2E node image
docker build -t clipshot-e2e-build -f tests/e2e/Dockerfile.e2e-node .
```

### Service / headless deploy

CLI supports service management via `clipshot service ...`.

Typical daemon startup shape:

```bash
clipshot daemon \
  --port 19231 \
  --http-port 15282 \
  --hub-url https://clipshot.cc \
  --group-token clip_xxx \
  --relay-url https://clipshot.cc/relay/
```

### Install flow

```bash
curl -fsSL https://clipshot.cc/install.sh | bash
```

`scripts/install.sh` flags:

- `--gui` - build with GUI/tray support (requires Tauri/WebKit deps)
- `--autostart` - enable autostart on login
- `--help` / `-h` - show usage

After install, pair with another device:

```bash
clipshot pair              # generate 6-digit code on first device
clipshot pair 482917       # enter code on second device
```

Optional: `clipshot setup` opens browser for Portal account (not required for pairing).

## Key Files

### Rust core

- `src/main.rs` - startup mode selection and CLI-vs-GUI/browser dispatch
- `src/lib.rs` - public module graph and classic `App` pipeline for SSH/OSC52 path
- `src/error.rs` - typed app errors and user-facing suggestions
- `src/content.rs` - canonical clipboard payload representation
- `src/paths.rs` - config/data/control-socket locations
- `src/logging.rs` - logging initialization
- `src/sync_status.rs` - global atomic sync state singleton

### GUI / Tauri

- `src/gui/tauri_app.rs` - Tauri setup, invoke handler list, hotkey worker, tray/dock wiring, test hooks
- `src/gui/commands.rs` - 38 Tauri command definitions (includes pair_generate, pair_join, pair_poll, pair_confirm, pair_abort)
- `src/gui/daemon_bridge.rs` - Tauri-facing async bridge around `AsyncDaemon`
- `src/gui/daemon_bridge_tick.rs` - 50ms tick loop, auto-resume, panic recovery, icon updates
- `src/gui/daemon_bridge_state.rs` - centralized icon-state projection
- `src/gui/transfer_progress.rs` - dock/taskbar progress + macOS sleep prevention
- `src/gui/menu_labels.rs` - tray/dock menu labels
- `src/gui/hotkey.rs` - native hotkey listener infrastructure
- `src/gui/hotkey_toggle_worker.rs` - queued hotkey/tray toggle serialization
- `src/gui/tray_animator.rs` - tray/dock icon animation logic
- `src/gui/paste_simulate.rs` - simulates Cmd+V / Ctrl+V paste via CGEvent (macOS) or enigo (Linux/Windows)
- `src/gui/dock_menu.rs` - macOS Dock context menu via `objc2-app-kit`
- `src/gui/config_change.rs` - config change detection and reactive handling (hotkey changed → update listener, hub changed → reconnect)
- `src/gui/daemon_bridge_lifecycle.rs` - lifecycle helpers (auto-resume on startup)
- `src/gui/display.rs` - graphical display detection
- `src/gui/hotkey_toggle_queue.rs` - hotkey press/release intent queue

### HTTP

- `src/http/server.rs` - all REST routes + CORS + SPA fallback
- `src/http/handlers.rs` - shared JSON/request helpers
- `src/http/handlers_peers.rs` - peer CRUD/connect routes
- `src/http/handlers_settings.rs` - settings routes
- `src/http/handlers_sync.rs` - sync control / transfers / clipboard routes
- `src/http/handlers_pair_v2.rs` - unified pair-v2 API (generate/join/poll/confirm/abort)
- `src/http/handlers_activity.rs` - status / activity / share URI routes
- `src/http/handlers_test.rs` - gated test routes
- `src/http/types.rs` - typed request models and response re-exports
- `src/http/static_files.rs` - embedded SPA/static asset serving

### Native QUIC sync (v0.7.3+)

- `src/p2p/v2/wire.rs` - postcard binary header format (ContentHeader, ControlMessage)
- `src/p2p/v2/sync_protocol.rs` - native QUIC send/receive (write_all/read_to_end), push_content_to_peer
- `src/p2p/v2/sync_handler.rs` - iroh ProtocolHandler for Router (ALPN: `clipshot/sync/1`)
- `src/p2p/v2/broadcast_queue.rs` - bounded FIFO send buffer (SRP, configurable 1-10 items)
- `src/p2p/v2/connection_monitor.rs` - connection lifecycle, RTT, sync timestamps, device limits (SRP)
- `src/p2p/v2/hooks.rs` - EndpointHooks for device limit enforcement
- `src/p2p/v2/connection_tracker.rs` - lightweight peer state tracker (replaces PeerManager)

### P2P daemon / center

- `src/p2p/v2/daemon/mod.rs` - `AsyncDaemon` struct and module split
- `src/p2p/v2/daemon/peer_snapshot.rs` - canonical API snapshot structs (FullPeerInfo, DaemonStatusFull)
- `src/p2p/v2/daemon/peer_connect.rs` - iroh endpoint startup, native QUIC connect, persistent iroh secret
- `src/p2p/v2/daemon/event_loop.rs` - daemon tick processing, broadcast_native, process_sync_received
- `src/p2p/v2/daemon/initializer.rs` - daemon startup initialization path
- `src/p2p/v2/uri.rs` - signed `clipshot://` URI format and verification
- `src/p2p/v2/uri_processor.rs` - URI add/probe path
- `src/p2p/v2/hub/` - portal WebSocket client and relay/subscription behavior
- `src/p2p/v2/peer_registry.rs` - configured-peer registry, address/node-ID mapping, RTT tracking
- `src/p2p/control/cache.rs` - global status/peer/share-uri/pro cache state
- `src/p2p/control/activity.rs` - global activity log + persistence
- `src/p2p/control/transfers.rs` - transfer tracking + unified history
- `src/p2p/control/discovery.rs` - `NetworkInterface` and `DiscoveredPeer` center types
- `src/p2p/control/settings.rs` - canonical `Settings` struct used across interfaces

### Settings / storage / sync

- `src/settings/mod.rs` - public settings store exports
- `src/settings/store.rs` - TOML store implementations
- `src/settings/validation.rs` - settings validation rules
- `src/sync/file_manager.rs` - writes synced content to disk and manages latest symlinks
- `src/sync/outbox.rs` - persistent catch-up outbox
- `src/sync/channel/` - sync channel implementations

### Clipboard / network

- `src/clipboard/macos.rs` - macOS clipboard path: `pngpaste` -> NSPasteboard PNG/TIFF -> `pbpaste`
- `src/clipboard/windows.rs` - Windows clipboard via hidden HWND + Win32 API (`windows-sys` crate)
- `src/clipboard/restore.rs` - restore clipboard content from sync path
- `src/network/discovery.rs` - mDNS browse/register logic
- `src/network/dns.rs` - address sorting and reachability filtering

### Web frontend

- `web/src/App.tsx` - routes, first-run shell, global hotkey hook
- `web/src/components/Layout.tsx` - main app chrome/header
- `web/src/components/Sidebar.tsx` - nav, Pro/Lite badge, pair entry point
- `web/src/components/SyncToggleButton.tsx` - icon-only header toggle button (Pause/Play/Loader2 icons); `SyncModeSelector.tsx` exists but is not currently used in production layout
- `web/src/components/PairDialog.tsx` - canonical pairing modal for all add-device paths
- `web/src/pages/OverviewPage.tsx` - status hero, recent activity, last synced card, node details
- `web/src/pages/PeersPage.tsx` - grouped peer list, filters, pinning, add/details flows
- `web/src/pages/HistoryPage.tsx` - unified timeline of transfers + events
- `web/src/pages/SettingsPage.tsx` - portal, general, sync, advanced, diagnostics settings UI
- `web/src/hooks/useDaemonStatus.ts` - status polling + immediate sync-state updates
- `web/src/hooks/usePeers.ts` - peer polling + disconnect grace stabilization
- `web/src/hooks/useHistory.ts` - adaptive history polling
- `web/src/hooks/useTransfers.ts` - 500ms transfer polling
- `web/src/hooks/useSettings.ts` - settings load/save/reset model
- `web/src/lib/api/endpoints.ts` - HTTP fallback implementation for Tauri commands
- `web/playwright.config.ts` - Playwright projects for browser, docker, relay, and VM

### Packaging / infra / scripts

- `Cargo.toml` - crate metadata, features, dependencies
- `tauri.conf.json` - Tauri packaging/runtime config
- `Dockerfile.headless` - server/headless image build
- `tests/e2e/docker-compose.api.yml` - docker mesh/browser API tests
- `tests/e2e/docker-compose.relay-e2e.yml` - relay E2E environment
- `tests/e2e/docker-compose.install.yml` - install flow E2E
- `scripts/build-release.sh` - release artifact build helper
- `scripts/remote-install.sh` - install/bootstrap helper
- `scripts/stress-test.sh` - large mesh stress harness

## Hub (Portal) Integration

Portal defaults and behavior:

- default hub URL: `https://clipshot.cc`
- settings fields:
  - `hub_url`
  - `group_token`
  - `relay_url`
  - `is_pro`
  - `node_password`
  - `central_relay_enabled`
  - `max_peers_free`

What the hub is used for:

- device discovery and group membership
- pairing code generation/join
- browser-based account setup (`POST /api/setup/sessions`, `GET /api/setup/sessions/:id`)
- subscription updates (`is_pro`, `central_relay_enabled`)
- device-limit feedback (`hub_limit_*` fields)
- relay configuration metadata
- Google OAuth login/registration

What the hub is **not**:

- not the clipboard data plane
- not the local HTTP API
- not the only connection path; manual / URI / mDNS still exist

UI surfaces driven by hub state:

- first-run flow depends on `group_token`
- `SettingsPage` shows portal connection + limit state + manage-devices link
- `Sidebar` shows Pro/Lite badge
- `OverviewPage` surfaces limit banners

## Settings Struct Field Reference

Complete `Settings` struct from `src/p2p/control/settings.rs`:

| Field | Type | Default | HTTP API field | Notes |
|---|---|---|---|---|
| `listen_port` | `u16` | `19231` | `listen_port` | |
| `auto_discover` | `bool` | `true` | `auto_discover` | serde(default) yields `false` on missing field; `Settings::default()` is `true` |
| `max_peers` | `usize` | `10` | `max_peers` | |
| `auto_sync` | `bool` | `true` | `auto_sync` / `sync_enabled` | |
| `poll_interval_ms` | `u64` | `500` | `sync_interval_ms` | API alias |
| `retry_attempts` | `usize` | `3` | - | not settable via HTTP |
| `timeout_ms` | `u64` | `5000` | - | not settable via HTTP |
| `catchup_limit` | `usize` | `5` | - | not settable via HTTP |
| `broadcast_queue_size` | `usize` | `1` | - | 1-10, clipboard items queued for broadcast; not settable via HTTP |
| `theme` | `String` | `"system"` | - | not settable via HTTP |
| `node_password` | `Option<String>` | `None` | - | hub-authoritative |
| `auto_start` | `bool` | `true` | `auto_start` | GUI toggles OS Login Items / .desktop autostart |
| `notifications` | `bool` | `true` | `notifications` | |
| `max_file_size_mb` | `u64` | `10` | `max_file_size_mb` | |
| `transfer_timeout_per_mb_ms` | `u64` | `500` | `transfer_timeout_per_mb_ms` | |
| `transfer_timeout_base_ms` | `u64` | `5000` | `transfer_timeout_base_ms` | |
| `chunk_timeout_secs` | `u64` | `120` | `chunk_timeout_secs` | |
| `enable_iroh` | `bool` | `true` | `enable_iroh` | |
| `use_browser_ui` | `bool` | `false` | `use_browser_ui` | |
| `hotkey` | `String` | `"cmd+shift+s"` (macOS) / `"ctrl+shift+s"` (other) | `hotkey` | Sync toggle hotkey |
| `clipboard_hotkey` | `Option<String>` | `None` | - | Not in Settings UI; set via `settings.toml` only |
| `paste_path_hotkey` | `Option<String>` | `"cmd+b"` (macOS) / `"ctrl+b"` (other) | - | Paste file path hotkey; enabled by default |
| `direct_send_threshold_mb` | `u64` | `1` | `direct_send_threshold_mb` | |
| `hub_url` | `Option<String>` | `Some("https://clipshot.cc")` | `hub_url` | |
| `group_token` | `Option<String>` | `None` | `group_token` | |
| `relay_url` | `Option<String>` | `None` | `relay_url` | |
| `cors_origins` | `Option<Vec<String>>` | `None` | - | not settable via HTTP |
| `is_pro` | `bool` | `false` | - | hub-authoritative |
| `max_peers_free` | `usize` | `3` | `max_peers_free` | |
| `central_relay_enabled` | `bool` | `false` | - | hub-authoritative; serde alias `relay_enabled` for migration |
| `peer_relay_enabled` | `bool` | `true` | `peer_relay_enabled` | peer relay (other nodes in group) — always enabled |
| `n0_relay_enabled` | `bool` | `true` | `n0_relay_enabled` | N0 public relay (iroh infrastructure) — always enabled |
| `sync_max_files` | `usize` | `100` | - | not settable via HTTP |
| `sync_max_age_days` | `u64` | `30` | - | not settable via HTTP |
| `sync_max_size_mb` | `u64` | `200` | - | not settable via HTTP |

## Share URI Format

Canonical format produced by the daemon/CLI:

```text
clipshot://node/<node_id>?name=<name>&addresses=<addr1>%2C<addr2>&exp=<unix_ts>&sig=<hmac>&password=true (optional)
```

Fields:

- `<node_id>` - node identity string
- `name` - display name
- `addresses` - comma-separated routable addresses, URL-encoded
- `exp` - expiration UNIX timestamp
- `sig` - truncated HMAC-SHA256 signature

Related CLI:

```bash
clipshot share-uri --port 19231 --ttl 1800
clipshot add-uri 'clipshot://node/...'
```

Related Tauri/HTTP:

- Tauri `export_node_url`
- Tauri `add_peer_from_uri`
- HTTP `GET /api/share_uri`
- HTTP `POST /api/add_peer_from_uri`

## Cargo Features

```toml
[features]
default = ["gui", "mdns-discovery", "iroh", "relay-server"]
gui = ["dep:tauri", "dep:tauri-build", "dep:rdev", "dep:tauri-plugin-updater", "dep:x11", "dep:enigo"]
mdns-discovery = ["dep:mdns-sd"]
iroh = ["dep:iroh"]
relay-server = ["dep:iroh-relay"]
pq = ["iroh/tls-aws-lc-rs"]  # Post-quantum key exchange (ML-KEM + X25519)
```

### Practical meaning

- `gui` - enables Tauri app, tray, native hotkeys, desktop integrations
- `mdns-discovery` - enables local `_clipshot._tcp.local.` browsing/registration
- `iroh` - enables the current live P2P transport path and listener
- `relay-server` - enables embedded iroh-relay server
- `pq` - post-quantum key exchange via aws-lc-rs (ML-KEM + X25519 hybrid)

## System Dependencies

| Platform | Dependency | Why |
|---|---|---|
| Linux X11 | `xclip` | clipboard access |
| Linux Wayland | `wl-clipboard` | clipboard access |
| Linux desktop | `libayatana-appindicator` / equivalent | tray support for Tauri on some distros |
| Linux pure headless | none GUI-specific | use browser/headless build |
| macOS | `pngpaste` | fast PNG clipboard extraction; otherwise fallback is NSPasteboard TIFF/PNG |
| macOS | built-in AppKit / NSPasteboard | native clipboard and dock integration |
| Windows | WebView2 Runtime | Tauri GUI rendering (Edge-based, pre-installed on Win10/11) |
| Windows | `windows-sys` crate | raw Win32 API for clipboard HWND, GlobalAlloc, SetClipboardData |
| Windows | `clipboard-win` crate | clipboard read (Unicode/Bitmap/FileList) |
| Windows | `rdev` crate | global keyboard hooks (WH_KEYBOARD_LL) for hotkeys |
| Linux/Windows | `enigo` crate | simulate Ctrl+V paste keystrokes in `paste_file_path` (gui feature) |
| All | `socket2` crate | sets `SO_KEEPALIVE` + per-OS TCP keepalive intervals (30/10/3) on the hub WebSocket socket; bounds half-open detection to <=60s on Linux after sleep/wake (v0.8.23 F2) |

### macOS clipboard specifics

`src/clipboard/macos.rs` reads in this order:

1. `pngpaste -` if available
2. `NSPasteboard` `public.png`
3. `NSPasteboard` `public.tiff` -> convert TIFF to PNG
4. files from `NSFilenamesPboardType`
5. text via `pbpaste` with timeout and UTF-8 locale env

## Build Artifacts

Release builds in `dist/`:

- `Clipshot_VERSION_aarch64.dmg` - macOS DMG installer (with Applications symlink)
- `Clipshot_VERSION_x64-setup.exe` - Windows NSIS installer
- `clipshot-macos-arm64` - macOS ARM bare binary
- `clipshot-macos-x64` - macOS Intel bare binary
- `clipshot-linux-x64` - Linux headless binary
- `clipshot-windows-x64.exe` - Windows bare binary

Supporting images/build outputs:

- `clipshot-headless` - headless daemon container
- `clipshot-e2e-build` - E2E test node image
- embedded frontend assets served by Tauri/browser mode and HTTP static handlers

## Distribution

Binaries are served from two sources (both must be in sync after each release):

| Source | URL | Usage |
|---|---|---|
| GitHub Releases | `github.com/axelbaumlisto/clipshot/releases/download/vX.Y.Z/FILE` | Landing page download buttons |
| clipshot.cc/dist | `clipshot.cc/dist/FILE` | `install.sh` default, direct links |

### clipshot.cc/dist

- Caddy route: `handle /dist/*` with `file_server { index off }`
- Host path: `/home/spex/app/bundle/clipshot-landing/landing/dist/`
- Container path: `/srv/static/dist/`
- Upload: `scp FILE spex:/home/spex/app/bundle/clipshot-landing/landing/dist/`

### install.sh

```bash
curl -fsSL https://clipshot.cc/install.sh | bash
```

Flags:
- `--code=CODE` - 6-digit pair code from existing device (optional, can pair after install)
- `--headless` - download bare binary only (no DMG/GUI)
- `--no-autostart` - skip systemd/launchd service creation
- `--port=PORT` - daemon port (default 19231)
- `--hub=URL` - hub URL (default https://clipshot.cc)
- `--dist=URL` - override download base URL

Default behavior:
- macOS: downloads DMG → mounts → copies .app to /Applications → `xattr -cr` (quarantine)
- macOS `--headless`: downloads bare binary to `~/.local/bin/`
- Linux: always downloads bare binary (no Linux GUI)

## Payments (Lemon Squeezy)

Payments processed by Lemon Squeezy (Merchant of Record).

| Product | Variant ID | Price | Type |
|---|---|---|---|
| Clipshot Pro | `1601525` | $5/month | Subscription |
| Clipshot Lifetime | `1601537` | $100 | One-time |

- Store ID: `314574`
- Store URL: `shotclip.lemonsqueezy.com`
- Webhook: `POST /api/webhooks/lemonsqueezy` on portal backend
- Webhook secret: in `docker-compose.yml` env `LEMONSQUEEZY_WEBHOOK_SECRET`
- Checkout flow: portal creates checkout URL via LS API → user pays → webhook fires → activation code emailed
- Activation: user enters code at `clipshot.cc/activate` or `clipshot setup`
- License keys: generated by LS, unlimited activations per key

## Legal Pages

| Page | URL | Source |
|---|---|---|
| Terms of Service | `clipshot.cc/terms` | `clipshot_portal/frontend/app/terms/page.tsx` |
| Privacy Policy | `clipshot.cc/privacy` | `clipshot_portal/frontend/app/privacy/page.tsx` |

Key TOS provisions:
- Pricing may change at any time (30-day notice for existing subscribers)
- Lifetime = lifetime of the product, not the purchaser
- Cloud services maintained while commercially viable (12-month shutdown notice)
- Features may move between tiers; core P2P sync always available
- P2P sync works without cloud services

## Windows Firewall

First launch triggers Windows Security dialog ("allow public/private networks").
For automation, pre-add rules:

```bash
netsh advfirewall firewall add rule name="clipshot" dir=in action=allow program="PATH\clipshot.exe" enable=yes
```
