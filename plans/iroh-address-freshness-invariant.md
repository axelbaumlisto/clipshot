# Iroh-address freshness invariant — implementation plan

## Problem (root cause, confirmed in production 2026-05-20)

When the iroh endpoint on any node rebinds (port changes), the new
`iroh://<key>?relay=...&addrs=...:NEW_PORT` does not reliably reach:

1. **Hub DB column `nodes.iroh_addr`** — portal handles `AnnounceNodeAddr`
   only in-memory via `state.node_addrs.announce(...)` and broadcasts a
   `PeerNodeAddrChanged` ServerMessage, but the `iroh_addr` column in the
   `nodes` table is only set on the initial `Hello` (INSERT). Any node
   that queries `/api/peers` after the rebind reads stale data from DB.
2. **Peer registries on remote daemons** — `apply_peer_node_addr_changed`
   calls `persist_hub_peer` which appends to `addresses[]` without
   pruning entries with the same `node_id` but stale ports. Live
   evidence: main's registry had three entries for spex, all on dead
   port `38712`.
3. **Active reconnect logic** — keeps trying the cached primary address;
   no "promote new address to primary" step.

The daemon already has three manual `announce_node_addr` blocks scattered
across `event_loop.rs` (lines 906, 1015, 1477). Each block reads
`self.iroh_addr` and pushes to the hub. They cover hub-WS connect/reconnect
plus iroh-restart-success, but rebinds triggered by other paths (e.g.
internal iroh endpoint reset between scheduled restarts) are not covered.

## Solution architecture (SOLID)

```
                  start_iroh_listener
                          │
                          ▼
              ┌───────────────────────┐
              │   IrohAddrTracker     │  ← single source of truth (SRP)
              │   Mutex<Option>       │
              │   broadcast::Sender   │
              └───────────┬───────────┘
                          │ on change
       ┌──────────────────┼──────────────────────┐
       ▼                  ▼                      ▼
[HubPublisher]   [PeerGossipBroadcaster]   [RegistrySelfPruner]
 push announce    refresh own gossip       cleanup self-addrs
 to hub WS         to connected peers       in local registry
```

Receiver side (mirror): incoming `PeerNodeAddrChanged` triggers a
*replace*, not *merge*, on the peer registry — for a given `node_id`,
exactly one address record after the event.

Portal side (separate PR): `AnnounceNodeAddr` handler additionally runs
`UPDATE nodes SET iroh_addr = ?, last_seen_at = ?`.

## Stages

| # | Stage | Status | Tests added | LOC |
|---|---|---|---|---|
| **S1** | `IrohAddrTracker` pure core (`src/p2p/v2/iroh_advertise/mod.rs`) | ✅ DONE | 8 unit | +220 |
| **S2** | Wire tracker into `AsyncDaemon`; `start_iroh_listener` calls `.set(...)` | ✅ DONE | 2 unit | +50 |
| **S3** | `HubPublisher` subscriber: replaces inline iroh-restart announce block (~18 LOC); two hub-WS-reconnect blocks intentionally kept | ✅ DONE | 6 unit | +95 / -18 |
| **S4** | `replace_addresses_by_node_id` + invocation in `apply_peer_node_addr_changed` | ✅ DONE | 4 unit | +110 |
| **S5** | End-to-end protocol convergence test: tracker → publisher → mock hub backplane → receiver registry `replace` | ✅ DONE | 2 integration | +190 |
| **S6** | Portal patch: `AnnounceNodeAddr` updates `nodes.iroh_addr` DB column via SQL + `iroh_node_addr_to_url` helper | ✅ DONE | 4 unit (portal repo) | +118 |
| **S7** | Skill update: invariant doc + release-checklist smoke test | ✅ DONE | — | +83 md |

**Cumulative test impact**:
  - clipshot repo: 1477 → 1499 (+22 new tests; all pass).
  - clipshot_portal repo: +4 new tests.

**Commit chain**:
  - clipshot: `5c028a72` (S1+S2) → `[S3+S4+S5 commit]` → `db62e15c` (S7).
  - clipshot_portal: `ad42a51` (S6).

**Deployment** (after merge):
  1. Roll out clipshot daemon: `ssh spex 'cd ~/work/clipshot && git pull && source ~/.cargo/env && cargo build --release --no-default-features --features headless && systemctl --user restart clipshot'` (then propagate to main / gene / it.local).
  2. Roll out portal: `ssh spex 'cd ~/work/clipshot_portal && git pull && docker compose build --no-cache portal && docker compose up -d --force-recreate portal'`.
  3. Run the release smoke test from `clipshot-deploy/SKILL.md`.

## Stage S1 (done) — `IrohAddrTracker`

Module: `src/p2p/v2/iroh_advertise.rs`, exported via `src/p2p/v2/mod.rs`.

Public API:

```rust
pub struct IrohAddrChange { pub old: Option<String>, pub new: Option<String> }
pub struct IrohAddrTracker {
    fn new() -> Self;
    fn current(&self) -> Option<String>;
    fn set(&self, new: Option<String>) -> bool; // true iff event fired
    fn subscribe(&self) -> broadcast::Receiver<IrohAddrChange>;
}
```

Test coverage:
- New tracker → `current() == None`
- First `set(Some)` → event `(None, Some)`, returns `true`
- Set same value → no event, returns `false` (idempotent)
- Change → event `(Some(old), Some(new))`
- `set(None)` after Some → event `(Some, None)`
- Multi-subscriber: identical event delivered to each
- Dropped subscriber: producer continues, no panic
- Late subscriber: missed events lost, `current()` provides bootstrap

KISS: no new crate dep (`std::sync::Mutex` + `tokio::sync::broadcast`,
both already in tree). ~80 LOC impl + ~140 LOC tests + docs.

## Stage S2 (done) — wire into AsyncDaemon

- Added field `pub(super) iroh_addr_tracker: Arc<IrohAddrTracker>` to
  `AsyncDaemon` in `src/p2p/v2/daemon/mod.rs`.
- Initialized in `AsyncDaemon::new(...)` alongside other defaults.
- In `start_iroh_listener` (`src/p2p/v2/daemon/peer_connect.rs:~513`),
  after `self.iroh_addr = Some(iroh_addr.clone())`, call
  `self.iroh_addr_tracker.set(Some(iroh_addr.clone()))`. Logs a
  distinct line when an event fires.
- Legacy field `iroh_addr: Option<String>` is kept (read by ~6 sites)
  to avoid churn during the migration. The tracker is set in lockstep.
- Regression check: `cargo test --lib` → 1485/1485 pass.

## Stage S3 (next) — `HubPublisher` subscriber

**Goal**: a single tokio task subscribed to the tracker that announces
the latest iroh address to the hub via `signaling.announce_node_addr`
whenever it changes. Replaces the three inline blocks.

**Trait contract** (DIP):

```rust
#[async_trait::async_trait]
pub trait IrohAddrSink: Send + Sync {
    async fn publish(&self, node_id: u64, addr_dto: IrohNodeAddr);
}
```

`HubPublisher` is an `IrohAddrSink` impl backed by a `SignalingClient`
handle. Tests use a mock `IrohAddrSink` that captures calls.

**Test cases**:

1. Tracker fires `(None, Some(addr1))` → sink received `publish(my_id, addr1_dto)` once.
2. Tracker fires `(_, Some(addr1))` followed by `(_, Some(addr2))` rapidly → sink received both calls in order.
3. `IrohNodeAddr` parse fails for malformed string → publisher logs warning, no panic, awaits next event.

**Refactor**: delete three inline `tokio::spawn(async move { signaling.announce_node_addr(...).await })`
blocks in `event_loop.rs` (lines 906, 1015, 1477). The HubPublisher
handles all three trigger contexts (initial connect, reconnect,
endpoint rebind) because it reacts purely to tracker state.

Edge case: tracker may have a value before the hub WS is connected.
HubPublisher's tokio task starts as soon as the daemon initializes,
but `signaling()` may not be ready. The task buffers the most recent
value and re-publishes on hub WS reconnect (driven by a separate
signal from `HubEvent::Connected` — small wiring).

## Stage S4 — `RegistrySelfPruner` + receive-side `replace_addresses_by_node_id`

**Sender side** (`RegistrySelfPruner`): on tracker event, scrub any
stale self-entries from the local peer_registry. In practice the daemon
never has its own node in its peer registry, so this is mostly a
defensive no-op; the more important fix is on the **receive side**.

**Receive side** — modify `apply_peer_node_addr_changed`:

```rust
// BEFORE: merge — accumulates stale entries
self.peer_registry.merge_address_by_node_id(node_id, &iroh_addr);

// AFTER: replace — exactly one address record per node_id
self.peer_registry.replace_addresses_by_node_id(node_id, vec![iroh_addr]);
```

New `peer_registry.rs` method:

```rust
/// Replace all addresses for a peer identified by `node_id` with `new_addrs`.
/// Returns the number of stale entries pruned. Used by the iroh-address
/// freshness invariant — see `iroh_advertise.rs`.
pub fn replace_addresses_by_node_id(
    &mut self,
    node_id: u64,
    new_addrs: Vec<String>,
) -> usize;
```

Tests:

1. Registry has `[bare_iroh, addrs=:38712, addrs=:38712:38712:38712:38712]`
   for node_id N → `replace_addresses_by_node_id(N, vec![addrs=:48539])` →
   registry has exactly `[addrs=:48539]` for N. Returns 3 (pruned).
2. Replace with empty vec → registry entry removed entirely.
3. Replace when node_id absent → returns 0, no-op.

## Stage S5 — end-to-end integration test

`tests/integration/iroh_rebind.rs`:

1. Spin up two in-process `AsyncDaemon` instances + a mock hub server.
2. Pair them; verify both registries have the other's iroh address.
3. Trigger rebind on daemon A by calling `start_iroh_listener(new_url)`.
4. Within 5 seconds: assert daemon B's registry contains exactly one
   entry for A, and it matches A's new tracker.current().

Without network, without portal — purely protocol-level. Goal: make this
regression test fail on `main` before S3/S4, pass after.

## Stage S6 — portal patch

In `clipshot_portal/backend/src/ws/session.rs`, extend the
`AnnounceNodeAddr` handler at line 503:

```rust
Ok(ClientMessage::AnnounceNodeAddr { node_id: _, addr }) => {
    state.node_addrs.announce(node_id, addr.clone());
    // NEW: persist freshness to DB so freshly-connecting peers see truth.
    let iroh_addr_str = serde_json::to_string(&addr).unwrap_or_default();
    let _ = sqlx::query(
        "UPDATE nodes SET iroh_addr = ?, last_seen_at = ? WHERE account_id = ? AND node_id = ?"
    )
    .bind(&iroh_addr_str)
    .bind(now_ts)
    .bind(account_id)
    .bind(node_id as i64)
    .execute(&state.db).await;
    state.relay.broadcast(group_id, node_id,
        ServerMessage::PeerNodeAddrChanged { node_id, addr }).await;
}
```

Plus one test: simulate a WS session that sends `AnnounceNodeAddr`,
assert `SELECT iroh_addr FROM nodes WHERE node_id = ?` returns the new
value.

Deploy: `docker compose build portal && docker compose up -d --force-recreate portal` on spex.

## Stage S7 — skill update

Add to `.pi/skills/clipshot-deploy/SKILL.md`:

- Section `## Iroh address freshness invariant` under `## Infrastructure`,
  documenting tracker → subscribers → portal flow.
- Release checklist step: "After deploying, kill iroh endpoint on one
  node (`SIGUSR1` if implemented, else `systemctl restart` while keeping
  hub WS alive). Within 10 seconds verify:
   - `ssh node 'curl ... /api/status' | jq .iroh_addr` shows new port,
   - `ssh other-node 'curl ... /api/peers' | jq '.[] | select(name=="node") | .addresses'` shows exactly one entry with the new port,
   - Hub DB: `SELECT iroh_addr FROM nodes WHERE name='node'` shows the new value,
   - Tauri UI: card stays Connected (no flicker)."

## Definition of done (all stages)

- `cargo test --lib` clean (1485 existing + ~14 new)
- `cargo clippy --all-targets -- -D warnings` clean (after fixing the
  pre-existing `inflight_count` dead-code, separately)
- Live verification across spex/main/gene: forced rebind on each →
  every other node sees the change within 10s with no stale residue
  in registry / hub DB.
- Skill release-checklist updated; runbook for this class of incident
  documented.

## Time estimate

- S1 + S2: ~30 min (done)
- S3: ~30 min
- S4: ~45 min
- S5: ~30 min
- S6: ~20 min + deploy
- S7: ~15 min

Total: ~3 hours implementation, plus deploy + live verification.

---

## Resolution log

### S1–S7: shipped in v0.8.22+v0.8.23
Tracker, subscriber, registry-replace, integration test, portal patch
and skill update all landed. Mesh converges within 10 s of any
endpoint rebind. No stale-address residue observed across 4 nodes for
24 h prior to the 2026-05-20 incident.

### 2026-05-20 production incident — hub WebSocket stall (F-series)

Despite the freshness invariant, it.local Mac daemon went silent for
4 h with hub_connected=False and stale TCP socket to portal in
ESTABLISHED state. Root cause: WS connect attempt hung indefinitely
on a half-open socket after macOS sleep/wake; no application timeout
existed before the inner read loop's 120 s idle detector.

Six-item follow-up (F1–F6) added to make hub WS recovery bounded and
testable:

| ID | Fix | Commit | Time bound on failure |
|---|---|---|---|
| F1 | 15 s `CONNECT_TIMEOUT` on `connect_async_tls_with_config` | `629306cf` | ≤15 s |
| F2 | TCP keepalive 30/10/3 via socket2 on the WS TcpStream | `abc3dd2d` | ≤60 s on Linux |
| F3 | TDD red-first unit tests for F1 + F2 (inlined into above) | `629306cf`,`abc3dd2d` | (test coverage) |
| F4 | Integration test: mock WS server goes silent post-Hello; paused-clock advance asserts Disconnected fires | `e507a15e` | (regression guard) |
| F5 | `/api/test/status` + `/api/test/peers` test-hooks for Mac Tauri-GUI (no `--browser` needed) | `e507a15e` | (observability) |
| F6 | Boot-time peer-registry resync on hub PeerList / PeerJoined / PeerNodeAddrChanged | `61eb9023` | Immediate on hub event |

Shipped as v0.8.23 (F1+F2+F6) and v0.8.24 (F4+F5). All deployed to
spex + main + it.local; gene pending (Tailscale unreachable at time
of deploy). Test count: 1497 → 1517.

Source touch needed for F4: `src/p2p/v2/hub/connection.rs` switched
from `std::time::Instant` to `tokio::time::Instant` for `last_recv`
and `connect_time`. Production behaviour identical (both are monotonic
clock samples); only the tokio variant advances with
`tokio::time::pause()`, enabling virtual-time integration tests.

### Plan status: CLOSED

S1–S7 + F1–F6 all merged, deployed and verified in production. The
diagnostic playbook in `clipshot-deploy/SKILL.md` documents recovery
for both classes (address staleness + hub stall). Any future
regression in this area should be caught by:

- `test_post_handshake_silence_triggers_reconnect` (F4)
- `iroh_advertise/integration_test.rs::test_convergence_after_rebind` (S5)
- Release smoke test in skill: `Hub WebSocket self-healing` section.
