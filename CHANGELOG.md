## v0.8.22 — 2026-05-17

### Peer relay connectivity (6 layers fixed)

Connection through embedded peer relays now works end-to-end. Previously,
behind-NAT nodes (`main`, `gene`) could not reach each other even when a
publicly-routable peer (`spex`) was online with `relay-server` enabled —
the relay address never made it from the publisher to the consumers.

- **L1 — Relay DNS pre-resolved at startup**: relay URL is resolved once on
  boot, so subsequent broadcasts never block on DNS even on Tailscale/VPN
  paths where the resolver is unreachable.
- **L2 — `headless` cargo feature includes `relay-server`**: every headless
  daemon (CLI / server) can now act as a peer relay. Previously only GUI
  builds shipped the relay, which broke server-as-relay topologies.
- **L3 — `persist_hub_peer` stores `relay_addresses`**: relay addresses
  advertised by other nodes via the hub are now persisted to the local
  peer registry instead of being dropped after the WS frame is decoded.
- **L4 — `persist_hub_peer` merges addresses** when a peer changes ports
  / IPs, so we don't lose the relay entry on the next hub broadcast.
- **L5 — `ping_peer_by_id` (no DNS, no relay lookup)**: HealthWorker now
  pings peers by `EndpointId` and reuses the iroh path cache, fixing
  health timeouts on Tailscale where DNS for `relay.clipshot.cc` is
  blocked.
- **L6 — Portal persists + broadcasts `relay_addresses`** (KEY FIX): the
  hub previously hardcoded `relay_addresses: vec![]` in every `PeerList` /
  `PeerJoined` message, so consumers never learned about the publisher's
  relay even though the publisher reported it correctly. Migration 018
  stores it; the WS layer now reads and broadcasts it.

End result: `spex ✓ direct`, `main ✓ via spex relay`, `gene ✓ via spex relay`.

### HealthWorker adaptive backoff

Offline peers (e.g. Windows VM turned off) used to be pinged every 30 s
forever — wasting CPU, flooding logs, and making the UI flap between ✓
and `timeout`. Health probes now back off after sustained failure:

| consecutive failures | next-probe wait |
|----------------------|-----------------|
| 0..3                 | 30 s (base)     |
| 3..10                | 5 min           |
| 10+                  | 15 min          |

A successful probe — or a hub `peer_joined` event for that node — resets
the counter, so a returning peer is rediscovered within one cycle
(≤ 30 s) instead of waiting out the long backoff window. The 1–2-failure
"flaky peer" guard is preserved by regression test.

### Other

- **macOS clipboard**: received files now write through `NSFilenamesPboardType`
  so Finder paste works for transfers, not just text/images.


## v0.8.21 — 2026-05-13

### Unified Device Removal
- **PeerRemoved P2P gossip** — remove a peer from any node, all connected peers learn via native QUIC
- **Hub RemoveDevice** — removal propagates through portal to all group members
- **PeerRemovedFromGroup** — distinct hub message for explicit removal (vs soft disconnect)
- **No gossip loop** — receivers remove locally without re-broadcasting
- **Portal dashboard delete** — "Remove" button deletes node + broadcasts to all peers
- **device_id canonical identity** — portal deduplicates by device_id, not node_id

### Security
- **SQL injection fix** — portal remove_device uses parameterized queries (was format!)

### Pair Flow Fixes
- **pair_confirm hub reconnect** — generator connects to hub after writing provisional token
- **reconnect_hub reads settings** — fills missing hub_url/token from settings.toml

### UI
- **Tray menu "Manage Account"** — replaces "Upgrade to Pro", checks is_pro from hub cache
- **Onboarding overlay** — 4-slide guide after first pair (Cmd+B comparison table)

### Documentation
- **Cmd+B / Ctrl+B** paste path hotkey fully documented
- **macOS permissions** — Input Monitoring + Accessibility requirements
- **Registration flow** — setup, OAuth, dashboard, link account, Free vs Pro
- **Quick Start Guide** — first section of USER_GUIDE

### Tests
- Rust: 1637, Vitest: 271, Portal: 35
- E2E: 23 scenarios (pair, sync, remove, portal, collisions, dedup)

## v0.8.2 — 2026-05-12

### Group Token Flow
- **group_token in PairOffer**: joiner adopts generator's token automatically
- **Deferred token write**: pair_generate holds token in memory, persists only on pair_confirm (prevents UI flash)
- **group_switch warning**: PairDialog shows yellow banner when joining a different group
- **mDNS carries group_token** in TXT record for LAN pairing
- **Hub without registration**: any non-empty token gets hub + relay access (anonymous group auto-created)

### Portal
- **link-group updates anonymous groups**: claims ownership on account registration
- **link-group creates group row**: dashboard shows group immediately after link
- **Setup page accepts existing token**: pre-fills from URL param
- **Dashboard token card**: prominent "YOUR GROUP TOKEN" with copy button after login

### UI Improvements
- **Loading spinner** in AppContent before settings resolve (no more WelcomeScreen flash)
- **Copy button** for group_token in Settings page
- **TransferIndicator** in header: shows active transfer progress (↑ img.png 439KB/1.2MB)
- **connection_type detection**: relay vs direct based on address pattern
- **device_type inference**: server/desktop heuristic from peer name
- **Transport labels**: "Relay · QUIC" / "Direct · QUIC" in peer details
- **Latency display**: shows "—" instead of "Measuring..." when unavailable

### Infrastructure
- **install.sh**: --code optional, --token flag for direct token setup, v0.8.2 banner
- **Windows NSIS installer**: 5.1MB installer with Start Menu shortcuts + uninstaller
- **broadcast_queue_size**: now settable via HTTP API (validation 1-10)
- **RTT consolidation**: ConnectionMonitor is single source of truth, 47 lines dead code removed

### Tests
- Rust: 1616 → 1634 (+18)
- Vitest: 247 → 262 (+15)
- Portal: 35

# Changelog

## [0.8.1] - 2026-05-12

### 🐛 Relay Reconnect Resilience
- Central relay (`relay.clipshot.cc`) is no longer permanently blacklisted after temporary downtime
- Relay connect attempts increased from 3 to 5 with exponential backoff (2s→4s→8s→16s = 30s window)
- Periodic health reset every 5 minutes ensures central relay is always retried
- Peer relays still blacklisted after repeated failures (prevents hot-loop on dead peer relays)

### ✅ Verification
- 1612 Rust tests pass, clippy clean
- RED→GREEN TDD: `test_reset_failures_clears_counter`, `test_connect_via_relay_does_not_blacklist_central_relay`

## [0.8.0] - 2026-05-11

### 🚀 Unified Pairing (pair-v2) — Breaking Change

Complete rewrite of the pairing subsystem. One numeric 6-digit code replaces all
previous pairing mechanisms (portal WORD-WORD-NN codes, LOCAL_WORD_NN LAN codes,
tabbed UI). Pairing no longer requires a Portal account — `group_token` is
generated locally at first pair and accounts can be linked later.

#### New pairing API surface
- `POST /api/pair/generate` → `{code, digits, expires_in_secs}`
- `POST /api/pair/join` → `{digits, peer_addr}`
- `POST /api/pair/abort` → best-effort cancellation after confirmation mismatch
- Tauri commands: `pair_generate`, `pair_join`, `pair_abort`
- CLI: `clipshot pair` / `clipshot pair 482917` (6-digit numeric)

#### New pairing architecture
- `src/pair_v2/` module tree: `code.rs`, `digits.rs`, `offer.rs`, `handshake.rs`, `broker.rs`, `runtime.rs`
- `PairBroker` races multiple discovery transports (mDNS + Portal `/api/pair/v2`)
- `PairDiscovery` trait with `InMemoryDiscovery`, `MdnsDiscovery`, `PortalDiscovery` implementations
- X25519 keypair + DH shared secret → 4-digit confirmation digits for MITM detection
- `ensure_group_token()` generates local UUID at pair time (account-decoupled onboarding)

#### Removed legacy pairing
- Deleted `src/pair.rs` and `src/http/handlers_local_pair.rs`
- Removed Tauri commands: `pair_generate_code`, `pair_join_code`, `pair_generate_local`, `pair_join_local`
- Removed HTTP routes: `/api/local-pair/generate`, `/api/local-pair/join`
- Removed tabbed PairDialog UI (Pair Code / Local Pair / Scan LAN tabs)
- Removed `joinOnly`, `defaultTab` props from PairDialog/AddPeerDialog
- Removed discover tab from AddPeerForm

#### UI changes
- PairDialog: 2-button menu (Generate code / Enter code) → generate/join/verify flow
- WelcomeScreen: primary CTA is "Connect your first 2 devices" (no create-account)
- SettingsPage: "Sign in to link this group" action when group_token exists
- AddPeerForm: flat layout with pair code + advanced section (paste URI / manual)

#### Portal side (clipshot_portal)
- `POST /api/pair/v2` + `GET /api/pair/v2/:code` (publish/resolve pair offers)
- `POST /api/account/link-group` + `GET /api/account/groups` (deferred account linking)
- Legacy `/api/pair` preserved for backward compatibility (pending removal window)

### 🐛 Resilience fixes
- **Iroh listener auto-restart**: detect socket closure reactively via `endpoint.is_closed()` instead of waiting for the 120s watchdog stall.
- **Cmd+B echo loop**: `paste_file_path` no longer triggers a duplicate broadcast of the same image.
- **mDNS self-discovery pollution**: address validator rejects loopback, unspecified, x.x.x.0 and port 0.
- **Peer registry one-time GC**: invalid legacy entries dropped at `PeerStore::load` time.
- **Peer-relay blacklist**: `find_peer_relay_url` skips known-dead relays.

### ✅ Verification
- Rust: `cargo test --lib` — 1603 passed, `cargo clippy --all-targets -- -D warnings` clean.
- Frontend: 247 vitest tests passed, `tsc --noEmit` clean.
- E2E: `pair-v2-portal-test.sh` passes (code exchange + digit parity + joiner connectivity).
- E2E: `pair-v2-lan-test.sh` SKIP (Docker mDNS multicast limitation).

## [0.7.6] - 2026-05-10

### 🚀 Connectivity Reliability
- Central `relay_url` is now stored separately from direct addresses and is never auto-cleared.
- Bare `iroh://<node>` addresses are rebuilt into full relay-aware connect URLs at connection time.
- Connection-refused cleanup now removes only stale direct addresses, preserving the central relay path.
- Removed the "hub down → strip relay" behavior that made relay peers disappear after transient hub outages.

### ⚡ Broadcast & UX
- Detached `BroadcastWorker` sends clipboard updates outside the tick loop, so slow peers no longer block local clipboard polling.
- Tray/Dock Sending state appears immediately on copy and returns to Idle after the first confirmed peer delivery.
- Broadcast uses live peer cache, including hub-discovered/transient peers.
- Large screenshots and files continue sending even when one relay peer is slow or temporarily unavailable.

### 🐛 Fixes
- Fixed spex/main/gene visibility loss caused by stale bare Iroh addresses.
- Fixed idle UI flicker where sync-proven devices disappeared after TTL despite the next relay sync succeeding.
- Fixed stuck Sending indicator when a slow relay peer never completed.
- Fixed reconnect state honesty: Online reflects real sync delivery instead of handshake-only success.
- Extended stale connection expiry to reduce idle UI flapping.

### ✅ Verification
- 1527 Rust lib tests passing.
- Clippy clean with `-D warnings`.
- Production test: macOS → gene/main/spex all confirmed via native QUIC relay/direct broadcast.

---

## [0.7.5] - 2026-05-09

### 🚀 Install Without Pair Code
- `curl -fsSL https://clipshot.cc/install.sh | bash` — works without `--code`
- First-time install opens browser for account creation (`clipshot setup`)
- After install shows unique pair code for adding second device
- Portal: Setup Sessions API (POST/GET /api/setup/sessions)

### 🔐 Pair Code Dictionary Expanded
- 64 → 256 adjectives, 64 → 256 nouns, 90 → 990 numbers
- **64.8M unique codes** (was 368K) — near-zero collision risk
- Zero overlap between adjective and noun word lists
- Retry on collision (PRIMARY KEY constraint)

### 🐛 Bug Fixes
- Relay fixed: container must be on `bundle_spex` Docker network
- Reconnect hot loop: `on_connected()` called after `connect_to_addr()`
- Canonical + full addr both registered in ConnectionMonitor
- Removed `WORD-WORD-00` placeholder from all user-facing docs

### 📚 Documentation
- Compact screenshots (1000×700), fresh annotations
- Deploy skill updated: Windows on main, dual landing paths, relay instructions
- install.sh shows `<YOUR-CODE>` not `WORD-WORD-00`

---

## [0.7.4] - 2026-05-09

### 🚀 Non-Blocking Clipboard Broadcast
- Clipboard stage no longer blocks on network I/O (was 1-15s per broadcast)
- New `BroadcastQueue` (SRP): bounded FIFO send buffer, configurable 1-10 items
- New `BroadcastPending` tick stage sends queued items in network phase
- 4 rapid screenshots → all 4 saved locally + all 4 delivered (zero loss)

### 🔧 Consistent Filenames Across Nodes
- Sender includes real filename (`img_YYYYMMDD_HASH.png`) in wire header
- Receiver uses sender's filename directly (was generating different name)
- Image content type preserved on receive (was converting Image → File)

### 🔐 Post-Quantum Key Exchange
- `cargo build --features pq` enables ML-KEM + X25519 hybrid via aws-lc-rs
- Default build uses ring (lighter, better cross-compile)

### ⚙️ New Setting: broadcast_queue_size
- Default: 1 (only latest clipboard item, previous dropped)
- Max: 10 (keep recent items in queue)
- Available in GUI Settings → Sync section

### 🐛 Bug Fixes
- Fixed reconnect hot loop (1700+ reconnects/sec blocking clipboard)
- Fixed ConnectionMonitor max_peers=3 blocking all outbound on gene
- Fixed gene visibility (was filtered as transient peer)

### 📚 Documentation
- Full CLAUDE.md audit: test counts, features, modules, settings, architecture decisions
- Screenshots retaken: settings (broadcast_queue_size), peers (QUIC), pair dialog (3 tabs)
- Token redacted from all screenshots

---

## [0.7.3] - 2026-05-08

### 🚀 Major: iroh 1.0 Migration + Native QUIC

Complete rewrite of the P2P transport layer. Replaced legacy chunked protocol
with native QUIC streams via iroh 1.0.0-rc.0.

#### Highlights
- **iroh 1.0.0-rc.0** — latest P2P networking library
- **Native QUIC sync** — `write_all`/`read_to_end` replaces 6000+ LOC of chunked transfer code
- **-10,000+ LOC removed** — deleted legacy transport, chunked transfer, PeerManager, AsyncPeer
- **20MB byte-exact** proven through 3-hop mesh chain (Docker) and production relay
- **Parallel broadcast** — all peers notified concurrently instead of sequentially
- **ConnectionTracker** — lightweight replacement for PeerManager (298 LOC vs 1856)

#### Transport
- Native QUIC streams via `clipshot/sync/1` ALPN protocol
- `wire.rs` — postcard binary header format (ContentHeader)
- `sync_protocol.rs` — send_content/receive_content/push_content_to_peer
- `sync_handler.rs` — iroh ProtocolHandler for Router
- Connection drop race fix in sync_handler (proper stream lifetime management)
- Broadcast timeout scales with content size (15s base + 5s per MB)
- Skip recently failed peers (5min cooldown) to avoid timeout waste

#### Security & Limits
- `EndpointHooks` — device limit enforcement on outbound connections via iroh hooks
- `ClipshotHooks` with `before_connect` policy (12 TDD tests)

#### Relay
- iroh-relay upgraded from 0.96 to 1.0.0-rc.0
- Caddy h1-only ALPN for relay.clipshot.cc WebSocket upgrade

#### Deleted (legacy)
- `iroh_transport.rs`, `iroh_factory.rs`, `handshake.rs`, `codec.rs` (protocol/)
- `chunked.rs` (transfer/)
- `content_handler.rs`, `content_send.rs`, `content_recv.rs`, `content_chunked.rs` (peer/)
- `failover.rs`, `probe.rs` (relay/)
- `connection_coordinator.rs` (daemon/)
- `manager.rs`, `broadcast_engine.rs`, `peer_index.rs`, `receive_loop.rs` (manager/)
- `AsyncPeer`, `PeerSender`, `PeerReceiver`, `PeerHealth` traits
- `converter.rs`, `dedup.rs` (peer/)
- `role.rs`, `factory.rs` (protocol/)
- `transport/` module (DataTransport, mock)
- `transfer/` module (BoundedQueue)
- `test_helpers.rs`
- Legacy integration tests (`iroh_integration.rs`, `iroh_stress_test.rs`)
- Dead Message variants: ChunkStart, ChunkProgress, ChunkComplete, ChunkCompleteAck, ContentWithMeta
- `process_accepted_peers`, `AcceptedPeer`, legacy health check

#### GUI Fixes
- **Latency measurement** — RTT from native QUIC broadcast time (not legacy Ping/Pong)
- **Peer names resolved** — iroh hex → friendly name via registry fallback search
- **"Unknown device"** eliminated from Overview, History, Peers
- **Tray icon stable** — no more NoPeers↔Idle flapping
- **"Last sync: just now"** — timestamp updated after each native sync
- **Device type** — removed misleading "Laptop" label when unknown
- **Transport badge** — shows "QUIC" instead of misleading "Direct"
- **Legacy settings hidden** — Transfer timeouts, Retry Attempts, Timeout (ms)
- **Addresses populated** — primary address shown when alternate list empty
- **Parallel broadcast** — 3 peers in 15s max (was 45s sequential)
- **Sync state** — uses native_connected_peers count (was always "no_peers")

#### Infrastructure
- iroh-relay 1.0 Docker image on spex
- systemd user service on main and gene
- Root daemon cleanup on main
- Stale peer cleanup across all nodes
- 20MB mesh forwarding: container (no internet) → main → gene byte-exact

#### Tests
- `native_sync_test.rs` — 3 integration tests (text, 1MB, bidirectional)
- `hooks.rs` — 12 TDD tests for EndpointHooks
- `connection_tracker.rs` — 7 TDD tests
- `wire.rs` — 5 ControlMessage serde tests
- Docker E2E: 3-node text sync + 20MB byte-exact verified

## [0.7.2] - 2026-05-05

- Native QUIC sync protocol (wire.rs, sync_protocol.rs, sync_handler.rs)
- Router + ProtocolHandler integration
- ConnectionCoordinator for duplicate prevention
- Streaming chunks (removed per-window ACK)
- Relay retry 3x with 2s delay
- Caddy h1-only ALPN fix for relay

## [0.7.1] - 2026-04-29

- device_id persistent identity
- effective_relay_url auto-derives from hub_url
- Portal device_id in PeerInfo broadcasts
- Zombie dedup in portal DB
- max_file_size sync on subscription
- Chunked transfer explicit failure (hash mismatch → ChunkComplete{hash=0})
