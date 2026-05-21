# v0.8.42 Architecture + iroh Reinvention Review

Generated 2026-05-22 by two parallel subagents (reviewer + researcher)
against commits 4afd03b4..e8882361 and the iroh 1.0.0-rc.0 docs.

Source reports (verbatim):
- `/tmp/clipshot-review-architecture.md` — SOLID/DRY/KISS audit
- `/tmp/clipshot-review-iroh.md` — iroh reinvention audit

This file is the consolidated executive view. Both source reports
should be folded into `plans/cleanup-iroh-and-legacy.md` when the
follow-up plan is drafted.

---

## Top blockers

### B1. `IrohPeerChannel` is dead code in production

`src/p2p/v2/daemon/initializer.rs:438-462` registers only `hub-ws`
and `iroh-relay` with `LivenessSupervisor`. The adapter at
`src/p2p/v2/transport_health/adapters/iroh_peer.rs` (≈220 LOC + 7
unit tests) compiles but never runs in the live daemon. Chaos
scenarios C1 (udp-blackhole-peer), C2 (sleep-wake) and C5
(multipath-confusion) — all marked "GREEN after T5" in the plan —
are not actually covered.

**Required:** either wire the adapter into supervisor with dynamic
per-peer registration (real T5 work), or feature-gate it to
`#[cfg(test)]` until wired.

### B2. Plan promised −600 LOC; reality is +1917 / −65

`git diff --shortstat c89ce316..d17e6d17 -- src/` shows the
transport-resilience series net **added 1852 LOC** to `src/`. The
LivenessSupervisor sits **alongside** the legacy machinery, not in
its place:

```
src/p2p/v2/daemon/tick/mod.rs:9-28          120s watchdog still present
src/p2p/v2/hub/connection.rs:37             CONNECT_TIMEOUT = 15s still there
src/p2p/v2/daemon/command_facade.rs:62-65   mark_iroh_listener_dead still called
src/p2p/v2/daemon/event_loop.rs:1328,1475,1538  3 reactive setters still there
```

Three writers, one `Arc<AtomicBool> iroh_listener_dead`, one reader
(the supervisor). The plan's "Definition of Done → net-negative LOC"
was not met.

---

## High-confidence iroh reinvention

| # | Our code | iroh upstream | LOC if replaced |
|---|---|---|---|
| C | `src/p2p/v2/iroh_advertise/` whole module (broadcast<IrohAddrChange>) | `Endpoint::watch_addr() -> Watcher<EndpointAddr>` (`endpoint.rs#1201`) | **−350** |
| E | `HealthWorker` 30s Ping/Pong = manual QUIC keepalive | `TransportConfig::keep_alive_interval` (we never set it; default = None) + `max_idle_timeout` (default 30s) | **−200** |
| A | per-peer Ping via `health_worker::probe_one` | `Connection::rtt()` + `Endpoint::remote_info().last_received` | **−150** (composes with E) |
| D | only log line `MultipathNotNegotiated`, no API | `Connection::paths() -> PathWatcher<PathInfoList>` (since 1.0-rc, PR #3899) | **+80** (replaces probes that A removes) |

Justified extensions (keep):
- B (Endpoint rebind): no public `Endpoint::rebind()` API; our watchdog
  may still be justified, but should be audited against production
  logs — if no recovery in 12 months, delete + use
  `Endpoint::network_change()` instead.
- F (peer registry): orthogonal to iroh `Discovery`; keep.
- H (Router supervision): iroh's `Router` does not auto-restart
  handler tasks on panic; our supervisor is the right layer.

---

## Behavioural regression introduced by e8882361

`src/clipboard/mod.rs:178-187`:

```rust
fn set_image_with_path_text(&self, _image_data: &[u8], path_text: &str) -> ... {
    // Default fallback: text only. macOS overrides for true multi-type write.
    self.set_content(&ClipContent::text(path_text))
}
```

On Linux desktop daemons (X11/Wayland with `has_display() == true`),
`sync_content` for Image content now **overwrites the user's clipboard
with text**, dropping the image bytes the user just copied. The
justification comment ("paste-as-image consumers are rare") is wrong
on Linux — Slack/Discord/Telegram desktop all consume image
clipboards.

**Fix options:**
1. Default impl: write image only, leave text alone (loses path-paste
   benefit on Linux but no regression).
2. Add `supports_multi_type_write: bool` capability to the trait;
   gate the policy on it.
3. Implement true multi-type write on `X11Clipboard` / `WaylandClipboard`
   (xclip + wl-copy both support MIME-typed writes).

---

## Cleanup queue, ranked by impact

| # | Refactor | LOC delta | Risk | Source |
|---|---|---|---|---|
| 1 | Set `endpoint.transport_config().keep_alive_interval = 10s, max_idle_timeout = 45s`; gut HealthWorker's active probe path; keep only the offline-peer backoff projector | **−200** | low | researcher §E |
| 2 | Delete `iroh_advertise/` entire module; subscribe to `endpoint.watch_addr().stream()`; hub publisher + registry pruner become two stream consumers | **−350** | low | researcher §C |
| 3 | Delete legacy: `tick/mod.rs` 120s watchdog block, `CONNECT_TIMEOUT` constant, `mark_iroh_listener_dead` ad-hoc callers. Have `IrohRelayChannel::probe` check `endpoint.is_closed()` directly | **−150** | medium | reviewer #5 |
| 4 | Replace HealthWorker probe path with `Connection::rtt()` + `RemoteInfo::last_received` projector | **−150** | low | researcher §A |
| 5 | Either wire `IrohPeerChannel` into supervisor with dynamic peer-registration, or feature-gate it `cfg(test)` | +50 or −220 | low | reviewer §1 |
| 6 | Delete `HubClientHealthHandle` wrapper; take `Arc<HubClient>` directly | **−35** | low | reviewer §4.2 |
| 7 | Fix Linux/Windows regression in `set_image_with_path_text` default impl (gate behind `supports_multi_type_write` capability or implement true multi-type on X11/Wayland) | +15 / 0 | high | reviewer §5.2 |
| 8 | Delete `HubClient::last_recv()` + `request_reconnect()` public methods (production-unused; only the handle uses them now) | **−20** | low | reviewer §3.2 |
| 9 | Inject `has_display` into `FileSyncManager` ctor instead of reading global env every sync; lets `test_sync_updates_clipboard_with_path` be strict | +15 | low | reviewer §5.5 |
| 10 | Wire `Connection::paths()` PathWatcher into `IrohPeerChannel` for real multipath diagnostics instead of log line | +80 | medium | researcher §D |
| 11 | Audit `iroh_listener_dead` watchdog against 12 months of production logs; if no recovery → delete and use `Endpoint::network_change()` | **−80 to −120** | medium | researcher §B |
| 12 | Expose `Endpoint::metrics()` via `/api/test/transport_health` (or new `/api/test/iroh_metrics`); stop duplicating QUIC counters in `SupervisorMetrics` | +30 | low | researcher §G |

**Aggregate if 1+2+3+4 applied: ~−850 LOC net deletion, simpler model.**

---

## Other findings

### From reviewer (smaller scope)

- **3.2 DRY**: `HubClient::last_recv` + `HubClientHealthHandle::last_recv`
  do the same job. Drop the handle.
- **4.1 KISS**: `SupervisedChannel::new(h, r, policy)` requires two
  trait-object clones for one struct. Blanket
  `trait HeartbeatAndRestartable` would save ~3 LOC per wire-up site
  (skip unless IrohPeerChannel wiring multiplies the call sites).
- **5.3 correctness**: `suppress_next_hash` is a single-shot global
  slot. Concurrent suppress writes overwrite each other; today's
  call sites are serial, so safe, but not defended by a test.
- **5.4 perf**: `get_settings_cache()` reads disk on every
  `sync_content` call. Rename to `load_settings_from_disk()` or
  actually cache.
- **5.6 lifecycle**: supervisor `JoinHandle` is stored but never
  aborted. On hub-URL settings change → second supervisor spawned,
  first leaks.
- **5.7 mild over-modelling**: `PasteboardSource::Image` and
  `Empty` variants are unreachable in `macos.rs::get_content` —
  the `match` only differentiates `Files` from "fall-through".

### From researcher (lower priority)

- **G Observability**: iroh has `Endpoint::metrics()` and
  `iroh_metrics::service::start_metrics_server` for OpenMetrics
  export. Our test-hooks endpoint duplicates some of the same
  counters. Cheap fix: also expose iroh's `EndpointMetrics`.
- **F Discovery**: low priority — could implement `Discovery` trait
  for our hub publisher so `endpoint.connect(node_id, ALPN)` resolves
  via hub natively, dropping ~100 LOC of `iroh://<hex>?addrs=...`
  string plumbing.
- **H Router**: minor refactor — split "rebuild Router" from
  "rebind Endpoint" so a panicking ALPN handler doesn't trash live
  connections.

---

## What this means for the v0.9.0 plan

`plans/transport-resilience-unified.md` should be amended:

- T2-T6: landed (in the sense that supervisor + traits + adapters +
  endpoint compile and run for hub-ws + iroh-relay)
- T7 (CI + soak): landed
- **T-deletion (implicit in T6): NOT landed** — legacy watchdog +
  CONNECT_TIMEOUT still present
- **T5-wire: NOT landed** — IrohPeerChannel not registered
- New item: **T-iroh-API-replacement** — items #1, #2, #4 above
- New item: **T-linux-fix** — item #7 above

Suggested follow-up plan name: `plans/cleanup-iroh-and-legacy.md`,
sequencing T-iroh-API-replacement first (largest deletion, lowest
risk), then T-deletion, then T-linux-fix, then optionally T-wire +
T-paths.

---

## Net architectural verdict

The clipboard series is clean. The transport-resilience series is
structurally sound (traits, policy/mechanism split, OCP for
adapters) but did not deliver its stated goals: the legacy machinery
was not deleted, one adapter is unwired, and three subsystems
(`iroh_advertise/`, `HealthWorker` probing, `iroh_listener_dead`
watchdog) duplicate iroh's public 1.0-rc API. Aggregate cleanup
opportunity: ~850 LOC of code we built and maintain that iroh now
gives us directly.
