# Transport Resilience Unified

## Why this plan exists

Today's production debugging session exposed the pattern we keep repeating:

| Date | Symptom | Reactive fix |
|---|---|---|
| 2026-05-19 | spex iroh socket died, watchdog only caught it at 120 s tick stall | `iroh_listener_dead` flag + accept-loop error detection (archived plan) |
| 2026-05-20 | it.local hub WS hung 4 h post sleep/wake | F1 (15 s connect timeout) + F2 (TCP keepalive) + F6 (registry resync) |
| 2026-05-20 (same day, 3 h later) | spex iroh transport silently died with `MultipathNotNegotiated`, mesh split | (would have been F7+F8+F9 — symmetric to F1+F2+F6, layer by layer) |

Each fix is layer-specific. Each adds its own timeout constant, its own
keepalive loop, its own restart path, its own test scaffolding. After
six F-items we have:

- `HealthWorker` — pings peers every 30 s, backoff schedule of its own
- `iroh_listener_dead` flag — set by accept loop + 120 s tick-stall watchdog
- `hub::connection.rs idle_timeout = 120 s` — application read silence
- F1 `CONNECT_TIMEOUT = 15 s` — connect-side specific
- F2 TCP keepalive 30/10/3 — kernel level, hub WS only
- F6 `apply_authoritative_iroh_addr` — fires only on hub PeerList events
- `command_router` reconnect logic — manually invoked
- 86 grep hits across `src/p2p/v2/` for the words
  `watchdog | dead | last_alive | idle_timeout | CONNECT_TIMEOUT | restart_iroh`

This is whack-a-mole. The next failure mode will require F10+F11+F12,
each with its own constants and tests. **The architectural primitive is
missing**: a single, layer-agnostic supervisor that says "is this I/O
channel making progress, and if not, restart it".

This plan deletes the per-layer ad-hoc work and replaces it with one
small abstraction validated by one Docker chaos suite.

---

## Architectural target

### Two traits + one supervisor

```rust
// src/p2p/v2/transport_health/mod.rs (new module)

/// Anything whose health we want to monitor. Self-describes liveness.
#[async_trait]
pub trait Heartbeat: Send + Sync {
    /// Identifier for logging / metrics (e.g. "hub-ws", "iroh-relay",
    /// "iroh-peer-spex"). Must be stable across the channel's lifetime.
    fn id(&self) -> &'static str;

    /// Bounded liveness check. MUST return within `probe_timeout` even
    /// if the underlying channel is wedged. Returning `Err` increments
    /// the supervisor's failure counter; returning `Ok` resets it.
    async fn probe(&self, probe_timeout: Duration) -> Result<(), HeartbeatError>;
}

#[async_trait]
pub trait Restartable: Send + Sync {
    /// Tear down the dead channel and start a fresh one. Idempotent.
    /// MUST itself respect the same probe_timeout — never blocks forever.
    async fn restart(&self, restart_timeout: Duration) -> Result<(), RestartError>;
}

/// Single configurable policy applied to ALL channels.
#[derive(Clone, Debug)]
pub struct Policy {
    pub tick_interval: Duration,                  // default 10 s
    pub probe_timeout: Duration,                  // default 5 s
    pub consecutive_failures_before_restart: u8,  // default 3
    pub cool_down_after_restart: Duration,        // default 30 s
}

pub struct LivenessSupervisor {
    channels: Vec<SupervisedChannel>, // Heartbeat + Restartable + policy
    metrics: Arc<SupervisorMetrics>,  // observable via /api/test/transport_health
}

impl LivenessSupervisor {
    pub fn spawn(self) -> tokio::task::JoinHandle<()> { /* … */ }
    pub fn metrics(&self) -> Arc<SupervisorMetrics> { … }
}
```

### Three concrete adapters

| Channel | `Heartbeat::probe` | `Restartable::restart` |
|---|---|---|
| `HubWsChannel` | send `Ping` frame, await `Pong` within `probe_timeout`; on existing `idle_timeout > probe_timeout` since `last_recv` return `Err` immediately | drop sender → inner reconnect loop wakes |
| `IrohRelayChannel` | `endpoint.remote_info(relay_node).last_alive` < `2 × probe_timeout`; otherwise `endpoint.dial_relay()` with timeout | `endpoint.set_relay_url(None) → set_relay_url(Some(url))` |
| `IrohPeerChannel` (one per registered peer) | `endpoint.remote_info(peer).last_alive`; on stale → native QUIC `Ping` via existing `HealthWorker::probe_one` (reuse, don't rewrite) | `peer_registry.mark_addr_stale(peer)` → next gossip tick re-resolves via hub |

The `HealthWorker`'s exponential backoff schedule (3 fails → 5 min, 10
fails → 15 min) is **preserved** but moved into `Policy` as
`backoff_levels: Vec<(u8, Duration)>` so every channel benefits.

### Observability (already half-built — F5 from yesterday)

`SupervisorMetrics` is a single `Arc<RwLock<HashMap<&'static str, ChannelHealth>>>`
exposed via the existing test-hook server:

```
GET /api/test/transport_health → {
  "hub-ws":            {alive: true,  consecutive_failures: 0, last_probe_ms_ago: 8345, restarts_total: 0},
  "iroh-relay":        {alive: true,  consecutive_failures: 0, last_probe_ms_ago: 9201, restarts_total: 2},
  "iroh-peer-spex":    {alive: false, consecutive_failures: 4, last_probe_ms_ago: 38120, restarts_total: 1},
  ...
}
```

This is the single diagnostic surface. Replaces ad-hoc `/api/status`
`hub_connected` boolean, `/api/peers` `latency_ms`, debug log scraping.
None of those go away (back-compat); they become projections of
`SupervisorMetrics`.

---

## What goes away (delete list, net-negative LOC)

| Removed | Lines | Replaced by |
|---|---|---|
| `tick/mod.rs` 120 s tick-stall watchdog for iroh listener | ~80 | `IrohRelayChannel::probe + restart` |
| `hub::connection.rs` `idle_timeout = 120 s` constant + manual `last_recv` tracking | ~30 | `HubWsChannel::probe` (probe is the heartbeat) |
| F1's `CONNECT_TIMEOUT = 15 s` as standalone constant | ~10 | `Policy.probe_timeout` |
| `HealthWorker::poll_loop` task | ~120 | absorbed into supervisor; **only `probe_one` kept** as `IrohPeerChannel::probe` impl |
| `command_router` reactive reconnect commands triggered from UI | ~60 | supervisor drives all restarts; UI only requests `restart_now` |
| F7/F8/F9 we were about to write | ~300 | never written |

Net target: **delete 600+ LOC, add ~400 LOC**, end up with one mental
model instead of six.

### What stays as defense-in-depth (justified DRY exception)

- **F2 TCP keepalive 30/10/3** — kernel-level signal, fires before any app
  layer can react. Cheap, OS-only, no maintenance burden. Keep.
- **F6 `apply_authoritative_iroh_addr`** — different concern (address
  freshness, not liveness). Keep as-is.
- **Iroh's own multipath/migration logic** — upstream library. Trust it,
  observe it via `remote_info()`, restart endpoint only when WE detect
  staleness it didn't recover from.

---

## E2E chaos test suite (Docker, the bit that was missing)

### New compose file

`tests/e2e/docker-compose.transport-chaos.yml` — 3 clipshot nodes + 1
relay + 1 portal. Every node:

- has `CLIPSHOT_TEST_HOOKS=1` (F5 endpoints)
- has `CAP_ADD: [NET_ADMIN]` so the scenario scripts can run
  `iptables` / `tc qdisc` from inside the container
- ships `iproute2` + `iptables` in `Dockerfile.e2e-node`

### Scenarios (each runs in <2 min, all share one bootstrap)

Bootstrap: bring up mesh, pair node1+node2+node3, wait for
`peers=2/2` from each via `/api/test/peers`.

| ID | Failure injection | What our supervisor must do | Pass criterion |
|---|---|---|---|
| **C1: udp-blackhole-peer** | `iptables -A INPUT -p udp -j DROP` on node1 for 60 s | `IrohPeerChannel` probes time out → after 3 failures restart endpoint (no fix — port allocation is upstream) | After undo, node2+3 see node1 Connected within 30 s |
| **C2: sleep-wake** | `docker pause node1` for 5 min, then `docker unpause` | All channels on node1 are silent during pause; on unpause, kernel + supervisor each detect within their own bound (kernel keepalive first, supervisor second) | node1 ↔ rest mesh restored within 30 s of unpause |
| **C3: relay-restart** | `docker restart clipshot-relay` mid-transfer | `IrohRelayChannel::probe` fails → restart re-dials new relay session; transferable data resumes | All nodes: `iroh-relay.alive=true` within 30 s of relay coming back; in-flight transfer either completes or fails-cleanly (no zombie) |
| **C4: hub-blackhole** | `iptables -A OUTPUT -p tcp --dport 443 -d portal -j DROP` on node1 for 60 s | `HubWsChannel::probe` times out → restart triggers reconnect loop (F1 absorbed into Policy.probe_timeout) | After undo, `hub-ws.alive=true` within 30 s; peer list re-synced via F6 |
| **C5: multipath-confusion** | `tc qdisc add netem loss 100%` on direct addrs to node1; relay path stays clean | Direct paths fail → iroh's own migration should fall back to relay; supervisor only intervenes if iroh wedges (the `MultipathNotNegotiated` case) | Within 30 s, transfers traverse relay path; supervisor restarts only if iroh wedges (asserted via `restarts_total` delta) |
| **C6: portal-restart** | `docker restart clipshot-portal` (the WS endpoint) | Hub goes down; supervisor restarts; on portal back-up, fresh announce | `hub-ws` reconnects within 30 s; portal DB shows fresh `last_seen_at` on all nodes |
| **C7: 24-h soak** (CI-only, scheduled) | No injection — just run the mesh for 24 h | `restarts_total` must remain at 0 for all channels | All channels stay alive for 24 h with zero supervisor restarts |

### Test driver

`tests/e2e/scripts/transport-chaos/run.sh`:

```bash
set -euo pipefail
SCENARIO=${1:?scenario-id required}

source ./lib/mesh.sh     # boot_3_node_mesh, wait_for_peers, ...
source ./lib/chaos.sh    # netem_drop, iptables_block, ...
source ./lib/assert.sh   # assert_within(timeout, predicate)

boot_3_node_mesh
wait_for_peers 2 2 2     # each node sees the other two

case "$SCENARIO" in
  C1) chaos_udp_blackhole node1 60
      sleep 90
      assert_within 30 'peers_all 2 2 2'                       ;;
  C2) docker pause node1; sleep 300; docker unpause node1
      assert_within 30 'peers_all 2 2 2'                       ;;
  C3) ...
esac

print_health_summary    # cats /api/test/transport_health for each
```

`run-all.sh` iterates C1..C6; C7 is scheduled separately.

### Why bash + curl, not Playwright

These scenarios test daemon transport, not UI. Playwright would add
browser, JS test harness, and serial scheduling overhead. Bash + curl
against existing `/api/test/*` endpoints (already F5-shipped) is faster,
debuggable, and parallelizable.

---

## Sequencing — strict TDD, every commit leaves tree green

| # | Step | Time | Tests after | Tree state |
|---|---|---|---|---|
| **T0** | Land the plan + skill update + retire archive ref | 30 min | 1517 | green |
| **T1** | Build `Dockerfile.e2e-node` with `iproute2 + iptables` + add `CAP_ADD: NET_ADMIN` to the chaos compose file. Write all 6 scenarios as bash scripts. **Run them: all 6 RED** because no supervisor yet | 3 h | 1517 lib (unchanged) + 6 e2e RED | green lib / RED e2e (expected) |
| **T2** | Define `Heartbeat` + `Restartable` traits + `LivenessSupervisor` skeleton + `SupervisorMetrics`. Unit tests use mock heartbeats (`tokio::time::pause` for virtual time, as in F4). No production wiring yet | 2 h | +25 unit tests, e2e still RED | green |
| **T3** | Implement `HubWsChannel: Heartbeat + Restartable`. Wire it into supervisor. Delete F1's bespoke `CONNECT_TIMEOUT` constant; delete `hub::connection.rs` standalone `idle_timeout` (replaced by `Policy.probe_timeout`). **C4 + C6 chaos scenarios go green** | 2 h | 1542+, C4+C6 green | green lib + 2/6 e2e |
| **T4** | Implement `IrohRelayChannel`. Delete tick-stall watchdog block for relay. **C3 chaos green**. Keep iroh's own multipath logic — supervisor only intervenes after consecutive_failures | 2 h | 1550+, C3 green | green lib + 3/6 e2e |
| **T5** | Implement `IrohPeerChannel` (one per registered peer). Reuse `HealthWorker::probe_one` as the probe impl; delete `HealthWorker::poll_loop` task. **C1 + C2 + C5 chaos green** | 3 h | 1560+, all 6 green | **all green** |
| **T6** | Expose `GET /api/test/transport_health` (the single observability surface). Update `clipshot-deploy/SKILL.md` diagnostic playbook to read from there instead of 4 different endpoints | 1 h | +5 tests | green |
| **T7** | Wire C7 24-h soak into CI as scheduled workflow (separate job, weekly cron). Document in `tests/e2e/README.md` | 1 h | (CI only) | green |
| **T8** | Release v0.9.0. Major bump because diagnostic surface changes are observable. Deploy spex+main+gene+it.local. Run all 6 chaos scenarios against the live deploy as final acceptance | 1 h | live mesh green | green |

**Total: ~15 h of focused TDD.**

Each commit message:
```
T<n>: <one-line summary>

What this commit contributes to plans/transport-resilience-unified.md.
Tests added/removed: <delta>. LOC delta: <delta>.
```

---

## Definition of Done

### Functional
- All 6 chaos scenarios pass in CI in <15 min total (parallel where independent)
- C7 (24 h soak) passes on the live spex+main+gene+it.local mesh
- Diff is net-negative LOC (deleted > added). Verified by `git diff --shortstat v0.8.24..v0.9.0`

### Architectural
- `grep -rn "watchdog\|idle_timeout\|CONNECT_TIMEOUT\|last_alive\|restart_iroh\|mark_iroh.*dead" src/`
  returns ≤ 10 hits, all inside `transport_health/` module. (Today: 86.)
- Adding a new I/O channel (e.g. future BLE transport, future
  WebTransport) requires implementing only `Heartbeat + Restartable` —
  no edits to the supervisor, no new global constants

### Testability
- Single mock `Heartbeat`/`Restartable` impl powers all unit + integration tests
- All E2E chaos scenarios use one bash driver, one assertion library
- Production debugging answers "is X channel alive" from one curl, not four

### Observability
- `GET /api/test/transport_health` is the canonical source on Mac (F5 reused)
- Same data appears on Linux nodes via `/api/test/transport_health` (gated by `CLIPSHOT_TEST_HOOKS=1` as today)
- `clipshot-deploy/SKILL.md` diagnostic playbook collapses 4 sections into 1

---

## What this plan EXPLICITLY rejects

- ❌ "Just add F7+F8+F9 like F1+F2+F6" — that's the whack-a-mole loop we're escaping
- ❌ "Add metrics, hope ops notices" — the daemon must self-heal; metrics are for diagnostics not recovery
- ❌ "Skip Docker E2E for now, run on production" — that's how iroh wedge escaped F4's coverage
- ❌ "One big commit for it all" — TDD means small steps; T0-T8 each leave green
- ❌ "Don't delete F1's CONNECT_TIMEOUT, keep it for safety" — DRY violation; replaced by `Policy.probe_timeout`
- ❌ "Skip the trait abstraction, just add another generic timer" — OCP violation; would need editing supervisor for each new transport
- ❌ "Make `Heartbeat` generic over `T: Probe`" — adds zero value, just SOLID-for-show. Concrete async-trait stays
- ❌ "Reuse `AddressProber`" — different concern (probing strings of addresses), don't conflate

## Out of scope (separate work, separate plans)

- Iroh upstream `MultipathNotNegotiated` root-cause investigation. We
  detect-and-restart around it; upstream may eventually fix
- Portal DB schema changes (already done in S6)
- UI redesign of the Peers page to reflect supervisor state (nice-to-have
  follow-up)
- Replacing portal WS with QUIC RPC (architectural axe, separate plan)

## Risks

| Risk | Mitigation |
|---|---|
| Removing `HealthWorker::poll_loop` deletes load-tested backoff schedule | `Policy.backoff_levels` copies the schedule verbatim; covered by existing 271 frontend tests + 1517 lib tests |
| Supervisor restart loop oscillation if probe + restart both fail repeatedly | `Policy.cool_down_after_restart = 30 s` enforces gap; metrics expose `restarts_total` and CI asserts ≤ 3 per scenario |
| Cross-platform diff: macOS launchctl env vs Linux systemd env for `CLIPSHOT_TEST_HOOKS` | Skill already documents `launchctl setenv CLIPSHOT_TEST_HOOKS 1` for Mac and systemd `Environment=` for Linux. No new platform work |
| Tauri-GUI mode binding tiny_http for `/api/test/transport_health` on Mac requires `CLIPSHOT_TEST_HOOKS=1`. End users have it off. | Acceptable — production users see `/api/status` projection; only ops needs the full surface |

## Acceptance test (Mac/spex/main/gene final check)

```bash
# Deploy v0.9.0 to all 4 nodes
# Run all 6 chaos scenarios against spex+main+gene mesh
cd tests/e2e
docker compose -f docker-compose.transport-chaos.yml up -d
./scripts/transport-chaos/run-all.sh
# Expected: 6 PASS, 0 FAIL, total <15 min

# Then trigger the same failures on the LIVE mesh (it.local + Linux 3)
# via:
ssh spex 'sudo iptables -A INPUT -p udp -j DROP'           # C1 live
sleep 90
ssh spex 'sudo iptables -F'
# Then within 30 s:
for n in spex main gene; do
  ssh "$n" 'curl -sf http://127.0.0.1:15282/api/test/transport_health' | \
    jq '. | to_entries[] | {channel: .key, alive: .value.alive, restarts: .value.restarts_total}'
done
curl -sf http://127.0.0.1:18181/api/test/transport_health   # Mac (it.local)
# Expected: every channel alive=true, restarts_total>=1 (we caused it)
```

Sign-off: when this command shows green across all 4 nodes, v0.9.0 ships.
