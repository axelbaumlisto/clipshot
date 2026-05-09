# Changelog

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
