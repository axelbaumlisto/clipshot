---
layout: default
title: API reference
nav_order: 9
---
Clipshot exposes a local HTTP API in browser mode and in daemon setups that enable the HTTP server.

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
| GET | `/` | – | embedded `index.html` |
| GET | `/index.html` | – | embedded `index.html` |
| GET | `/assets/*` | – | embedded static asset or `404 text/plain` |
| GET | `/health` | – | `{"status":"ok"}` |
| GET | `/api/icon.png` | – | PNG chosen from current global sync state |
| GET | `/api/permissions` | – | `ApiResponse<PermissionInfo[]>` (macOS Input Monitoring etc.) |
| OPTIONS | any API path | – | `204` + CORS headers |

### Status / activity / share routes

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/api/status` | – | `ApiResponse<DaemonStatusFull>` |
| GET | `/api/activity` | – | `ApiResponse<ActivityEvent[]>` |
| GET | `/api/share_uri` | – | `ApiResponse<{ uri, node_id, expires_in_secs }>` or error if no routable addresses |
| GET | `/api/gossip_peers` | – | same payload as `/api/peers` |
| GET | `/api/transfers` | – | `ApiResponse<TransferInfo[]>` |
| GET | `/api/history?filter=<opt>&limit=<opt>` | – | `ApiResponse<HistoryEntry[]>`, default `limit=100` |
| GET | `/api/history/:id/content` | – | raw file bytes; `image/*` if history entry is image, else `text/plain; charset=utf-8` |
| GET | `/api/last_sync_path` | – | last sync file path |

### Peer routes

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/api/peers` | – | `ApiResponse<FullPeerInfo[]>` |
| POST | `/api/peers` | `{"name":"...","address":"..."}` | queues `PeerCommand::Add` |
| PUT | `/api/peers` | `{"address":"...","name"?:string,"addresses"?:string[],"auth_code"?:string|null}` | queues `PeerCommand::Update` |
| DELETE | `/api/peers/:name` | – | queues remove by peer name or stable hex node ID |
| POST | `/api/peers/:name/connect` | – | queues `ConnectByName` |
| POST | `/api/peers/:name/activate` | – | queues `ActivateByName` |
| POST | `/api/peers/:name/deactivate` | – | queues `DeactivateByName` |
| POST | `/api/peers/:name/reconnect` | – | queues `Reconnect` |
| POST | `/api/add_peer_from_uri` | `{"uri":"clipshot://..."}` | queues `AddFromUri` |

Validation details:

- `AddPeerRequest.name`: 1..64 chars
- `AddPeerRequest.address`: 7..255 chars
- `UpdatePeerRequest.address`: 1..255 chars
- `AddPeerFromUriRequest.uri`: 1..2048 chars, plus non-blank after trim

### Settings routes

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/api/settings` | – | `ApiResponse<Settings>` from daemon-authoritative projection |
| POST | `/api/settings` | partial settings update body | queues `UpdateSettings` |

Accepted update fields in `POST /api/settings`:

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
- `central_relay_enabled` (hub-authoritative; serde alias `relay_enabled`)
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
| POST | `/api/sync/pause` | – | queues `PauseSync` |
| POST | `/api/sync/resume` | – | queues `ResumeSync` |
| POST | `/api/sync/toggle` | – | queues `ToggleSync` |

Validation:

- `RetryTransferRequest.file_path`: 1..4096 chars
- `CopyToClipboardRequest.text`: 1..1_000_000 chars

### Pair routes (pair-v2, `src/http/handlers_pair_v2.rs`)

Unified pairing API for GUI, browser UI, and CLI. Code is a 6-digit numeric string.

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/api/pair/generate` | – | `{code, digits: null, expires_in_secs}` |
| POST | `/api/pair/join` | `{code}` | `{digits, peer_addr, joiner_eph_pub_hex}` |
| POST | `/api/pair/poll` | `{code}` | `{ready, joiner_eph_pub_hex, joiner_iroh_addr}` |
| POST | `/api/pair/confirm` | `{code, joiner_eph_pub_hex, joiner_iroh_addr?}` | `{digits}` |
| POST | `/api/pair/abort` | `{code}` | `{aborted, code}` |

Notes:
- `code` is a numeric 6-digit string (`^[0-9]{6}$`)
- `generate` stores an ephemeral X25519 keypair and returns `digits: null` (unknown until joiner arrives)
- `join` performs X25519 DH, pushes joiner eph_pub to portal, adds generator as peer
- `poll` is called by generator to detect joiner arrival via portal
- `confirm` computes DH with joiner eph_pub, derives 4 confirmation digits, adds joiner as peer (reciprocal)
- `abort` cleans up local keypair; subsequent `confirm` returns 404

### Sync / transfer / paste routes

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/api/paste_path` | – | saves clipboard to sync dir, writes path, simulates paste |

### Test-only daemon HTTP routes (`src/http/handlers_test.rs`)

These require `CLIPSHOT_TEST_HOOKS=1` or return `403`.

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/api/test/clipboard` | – | clipboard snapshot: text returns `type/data/size`, binary returns `type/size` |
| POST | `/api/test/reset` | optional `{"restore_peer_name"?,"restore_peer_addr"?,"max_peers_free"?}` | queues reset-state command |
| POST | `/api/test/kill_iroh_listener` | – | kills the active iroh listener (forces iroh restart on next tick) |

### Tauri hotkey test-hook HTTP server

This is a **separate** tiny_http server started from `src/gui/tauri_app.rs` only when `CLIPSHOT_TEST_HOOKS=1`.

Defaults:

- bind: `127.0.0.1:${CLIPSHOT_TEST_HOOKS_PORT:-18181}`

Routes:

| Method | Path | Body | Response |
|---|---|---|---|
| GET / POST | `/api/test/health` | – | daemon readiness (`200 ready` or `503 starting`) |
| GET / POST | `/api/test/sync_enabled` | – | current `sync_enabled` bool |
| GET / POST | `/api/test/hotkey_metrics` | – | queue/drain latency metrics |
| GET | `/api/test/native_listener_health` | – | native hotkey listener health |
| GET | `/api/test/icon_state` | – | `{ state, state_raw }` from global sync atomic |
| GET | `/api/test/native_key_events` | – | captured native key event snapshot |
| POST | `/api/test/native_key_events` | – | clears captured native key events |
| POST | `/api/test/hotkey_toggle` | optional `X-Clipshot-Source` header | emits `hotkey-toggle-sync` into app event loop |

### Setup sessions (device auth flow)

| Method | Path | Body | Response |
|---|---|---|---|
| POST | `/api/setup/sessions` | – | `{ session_id, url, expires_in }` |
| GET | `/api/setup/sessions/:id` | – | `{ status: "pending"\|"completed"\|"expired", group_token? }` |
| POST | `/api/setup/sessions/:id/complete` | `{ group_token }` (requires JWT) | `200` |

Used by `curl | bash` install script and GUI "Create Account" button. Session expires in 10 minutes.

### OAuth

| Method | Path | Notes |
|---|---|---|
| GET | `/api/auth/google` | Redirect to Google OAuth. Optional `?state=SESSION_ID` for device auth. |
| GET | `/api/auth/google/callback` | Google callback — creates/finds user, issues JWT. |
| GET | `/api/auth/github` | Redirect to GitHub OAuth. |
| GET | `/api/auth/github/callback` | GitHub callback. |

**Note:** OAuth and setup session routes above are **Portal-side** endpoints (clipshot.cc), not local daemon HTTP API routes. The local daemon serves routes like `/api/status`, `/api/peers`, etc.
