# Clipshot transport-resilience vs. iroh upstream — wheel-reinvention audit

Target version: **iroh `1.0.0-rc.0`** (Cargo.toml:82). All citations are against `docs.rs/iroh/latest/...` which currently serves the 1.0-rc docs.

---

## 1. Executive summary

**High-confidence reinvention (replace with iroh API):**

| Q | What we built | iroh equivalent | Confidence |
|---|---|---|---|
| **C** | `IrohAddrTracker` (broadcast<IrohAddrChange>) | `Endpoint::watch_addr() -> Watcher<EndpointAddr>` | **Very high** — drop-in replacement |
| **E** | `HealthWorker` running 30 s Ping/Pong as a de-facto keepalive | `quinn::TransportConfig::keep_alive_interval` / `max_idle_timeout` configured on `Endpoint::builder` | **Very high** — we never set these; iroh defaults are "keepalive off, 30 s idle" |
| **A** | per-peer Ping/Pong + `HealthState` for live peers | `Connection::rtt()`, `Connection::stats()`, `Endpoint::remote_info().last_received` | **High** for connected peers; medium for the "is offline peer back" case |
| **D** | log-only `MultipathNotNegotiated` line | `Connection::paths() -> PathWatcher<PathInfoList>` with `PathInfo::is_selected/is_closed/stats` | **High** |
| **G** | bespoke `SupervisorMetrics` + `/api/test/transport_health` | `Endpoint::metrics() -> EndpointMetrics` (OpenMetrics/Prometheus ready) | **Medium** — complementary, but we duplicate counters |

**Justified extension (keep):**

- **B** Endpoint rebind: we have a watchdog (`iroh_listener_dead`) that forces a full `start_iroh_listener` rebuild. iroh self-heals UDP socket churn internally but exposes no public "respawn" hook; our watchdog is justified *iff* we have evidence of a dead-socket scenario iroh doesn't recover from. Otherwise it is **redundant** — see §2.B.
- **F** Peer registry: iroh's `Discovery` trait resolves NodeId → addresses. Our `peer_registry.rs` adds human-name, RTT cache, transfer history, blacklist — **orthogonal** application concern. Keep.
- **H** Router supervision: `iroh::protocol::Router` *does not auto-restart* handler tasks on panic; it propagates `JoinError` on `shutdown()`. The supervisor-driven restart layer for ALPN handlers is **justified**.

**Bottom line:** ~3 sub-systems (`iroh_advertise/` ~250 LOC, the keepalive part of `health_worker.rs` ~150 LOC, the `iroh_listener_dead` watchdog ~80 LOC) are demonstrably reinventions of iroh's public API. Removing them and wiring the iroh primitives directly saves ~400–600 LOC of code we currently maintain and test.

---

## 2. Per-question analysis

### A. Endpoint health / liveness / RTT — **REINVENTION (partial)**

**iroh upstream:** **Yes**, multiple surfaces.

- `Connection::rtt() -> Duration` — "Current best estimate of this connection's latency (round-trip-time)" — [docs.rs/iroh/latest/iroh/endpoint/struct.Connection.html](https://docs.rs/iroh/latest/iroh/endpoint/struct.Connection.html) (Source `connection.rs#970-972`)
- `Connection::stats() -> ConnectionStats` — "Returns connection statistics." — same page (Source `connection.rs#976-978`); details in [docs.rs/iroh/latest/iroh/endpoint/struct.ConnectionStats.html](https://docs.rs/iroh/latest/iroh/endpoint/struct.ConnectionStats.html)
- `Endpoint::remote_info(node_id) -> Option<RemoteInfo>` — "Returns addressing information about a recently used remote endpoint…" — [docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html#method.remote_info](https://docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html) (Source `endpoint.rs#1543-1548`)
- `RemoteInfo` exposes per-transport `last_received` / `last_alive` via `TransportAddrInfo::usage()` — [docs.rs/iroh/latest/iroh/endpoint/struct.RemoteInfo.html](https://docs.rs/iroh/latest/iroh/endpoint/struct.RemoteInfo.html) and refactor history in [commit 9d06888](https://github.com/n0-computer/iroh/commit/9d068886f1c16d6a47ac3ce1c454369b00cd6de7) ("`RemoteInfo.last_received`, `last_alive`, `latency`").

**Our equivalent:** `src/p2p/v2/health_worker.rs` — 30 s wall-clock Ping/Pong via `sync_protocol::ping_peer_by_id` with custom 3/10-fail backoff (`BACKOFF_FAILS_MEDIUM`, `BACKOFF_FAILS_LONG` lines 41–45); `HealthState` per-peer failure counter (lines 51–116).

**Verdict:** **Reinvention for connected peers.** For any peer where we already hold (or can `endpoint.connect`) a `Connection`, `connection.rtt()` + `connection.stats()` answers "is this alive, how fast" without sending a single byte. For known-but-disconnected peers, `endpoint.remote_info(node_id)` gives us the last-seen timestamp without spawning a probe at all.

The only piece worth keeping from `HealthWorker` is the **policy** that "after 10 fails, back off to 15 min" — that is application policy, not transport mechanics.

**LOC savings if replaced:** drop the active-probe path. `health_worker.rs` shrinks from ~470 lines to a ~100-line "consult iroh remote_info, apply backoff policy" projector. **Net −300 to −350 LOC.**

---

### B. Endpoint rebind / restart — **LIKELY REDUNDANT (no public iroh hook, but iroh self-heals)**

**iroh upstream:** **No explicit `Endpoint::rebind()` or `respawn_socket()`.** What exists:

- `Endpoint::network_change()` — "Notifies the system of potential network changes. […] Even when the network did not change, or iroh was already able to detect the network change itself, there is no harm in calling this function." — [docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html](https://docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html) (Source `endpoint.rs#1564-1570`)
- `Endpoint::watch_addr()` — observes downstream effect of any re-bind iroh does internally (Source `endpoint.rs#1201-1215`).
- `Endpoint::online()` — re-await reachability after a network event (Source `endpoint.rs#1284-1303`).

iroh's design intent is that the **endpoint internally manages UDP socket lifecycle** and the application **never tears down/rebinds**. The 1.0 docs do not document any "endpoint died, rebuild me" public API.

**Our equivalent:**
- `src/p2p/v2/daemon/mod.rs`: `iroh_listener_dead: Arc<AtomicBool>` watchdog flag
- `src/p2p/v2/daemon/peer_connect.rs::start_iroh_listener` lines 355–545 — full rebuild of `iroh::Endpoint::builder(...).bind()` + `Router::builder(...).spawn()` from scratch.

**Verdict:** **Likely reinvention / over-engineering.** Iroh's stance is that you don't restart the endpoint; you call `network_change()` if the OS told you something changed, then keep using the same `Endpoint`. Killing the endpoint and re-binding loses all open `Connection`s, the relay handshake state, and the path-watcher subscriptions — a net negative unless the endpoint is actually wedged.

**Justified only if** the codebase has a documented production incident where `Endpoint` became permanently unable to send/receive and `network_change()` did not recover it. The plan doc (`plans/transport-resilience-unified.md`) appears to predate the 1.0-rc API.

**LOC savings if removed:** the `iroh_listener_dead` watchdog logic + supervisor's `IrohRelayChannel::restart` adapter ≈ **−80 to −120 LOC** plus simpler `start_iroh_listener` (the rebuild path doesn't need to be idempotent).

**Action:** before removing, search logs for cases where the watchdog *actually* recovered something. If none in 12 months → delete.

---

### C. Address advertisement / freshness — **REINVENTION (very high confidence)**

**iroh upstream:** **Yes — exact match.**

- `Endpoint::watch_addr() -> Watcher<EndpointAddr>` — "Returns a Watcher for the current EndpointAddr […]. The EndpointAddr will change as network conditions change, the endpoint connects to a relay server, the endpoint changes its preferred relay server, more addresses are discovered for this endpoint." — [docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html#method.watch_addr](https://docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html) (Source `endpoint.rs#1201-1215`).
- Built on the `Watchable` infrastructure introduced in [PR #2806](https://github.com/n0-computer/iroh/pull/2806) — "Add a Watchable struct for use in the Endpoint API".
- Usage pattern: `endpoint.watch_addr().stream()` yields every change; `endpoint.watch_addr().get()` reads current value.

**Our equivalent:** `src/p2p/v2/iroh_advertise/mod.rs` — entire module:
- `IrohAddrTracker { current: Mutex<Option<String>>, tx: broadcast::Sender<IrohAddrChange> }` (lines 60–66)
- `set()` does snapshot-and-swap + broadcast (lines 84–105)
- `subscribe()` returns a `broadcast::Receiver<IrohAddrChange>` (lines 109–111)
- + `hub_publisher.rs` consumer + `integration_test.rs`

This is literally a hand-rolled `tokio::sync::watch`/Watchable for a single value that **iroh already publishes on the same endpoint**.

**Verdict:** **Reinvention.** Direct replacement:
```rust
let mut stream = endpoint.watch_addr().stream();
while let Some(addr) = stream.next().await {
    hub_publisher::publish(&addr).await;
}
```

**LOC savings:** `iroh_advertise/mod.rs` (210 LOC) + `iroh_advertise/integration_test.rs` (~150 LOC) + tracker plumbing in `daemon/peer_connect.rs:514-522` + `daemon/mod.rs` field. **Net −300 to −400 LOC and one whole module.** Bug surface (the "spex port 48539 wasn't published" production bug the module was created to fix) becomes iroh's problem.

---

### D. Multipath / relay-vs-direct path observation — **REINVENTION OF DIAGNOSIS (high confidence)**

**iroh upstream:** **Yes, since 1.0-rc.**

- `Connection::paths() -> PathWatcher<PathInfoList>` — "Returns a Watcher for the network paths of this connection. A connection can have several network paths to the remote endpoint, commonly there will be a path via the relay server and a holepunched path." — [docs.rs/iroh/latest/iroh/endpoint/struct.Connection.html#method.paths](https://docs.rs/iroh/latest/iroh/endpoint/struct.Connection.html) (Source `connection.rs#1077-1110`)
- `PathInfo::is_selected()`, `PathInfo::is_closed()`, `PathInfo::stats()` — same page.
- Closed paths remain in the list — [PR #3899](https://github.com/n0-computer/iroh/pull/3899) ("Retain stats for closed and abandoned paths in the path watcher").
- Roadmap: full QUIC-MULTIPATH integration — [iroh.computer/blog/iroh-on-QUIC-multipath](https://iroh.computer/blog/iroh-on-QUIC-multipath).

The old `Endpoint::conn_type` / `ConnTypeStream` was removed ([commit 2f924d9](https://github.com/n0-computer/iroh/commit/2f924d90f903e029045432d7762db976934ccd8c)) and replaced by `Connection::paths()`.

**Our equivalent:** We **don't have a path-state surface at all**. The codebase only logs `MultipathNotNegotiated` and otherwise treats the connection as opaque. The `HealthWorker` Ping/Pong indirectly probes whichever path iroh chose, but we can't tell which one or detect a wedged-path-without-failover.

**Verdict:** **Reinvention-by-omission.** We're paying the cost of *not* using `Connection::paths()` — we ship logs that say "multipath not negotiated" with no follow-up because there's no API hookup. Wiring `path_watcher.stream()` into the supervisor's `IrohPeerChannel` adapter would replace ad-hoc logging with structured state ("relay path selected", "direct path closed at T+42 s", "switching").

**LOC delta:** small **add** (+50–80 LOC for a `PathWatcher` consumer), but removes the rationale for some of the per-peer health workers (D answers "is path wedged?" better than Ping/Pong).

---

### E. Per-connection idle timeout / keepalive — **REINVENTION (high confidence — we never set the obvious config)**

**iroh upstream:** **Yes.**

- `iroh_quinn::TransportConfig::max_idle_timeout` — default **30 s**. — [docs.rs/iroh-quinn/latest/iroh_quinn/struct.TransportConfig.html](https://docs.rs/iroh-quinn/latest/iroh_quinn/struct.TransportConfig.html)
- `iroh_quinn::TransportConfig::keep_alive_interval` — default **None** (disabled). "When set, will be sent at this interval during inactivity to prevent timeouts. Must be lower than max_idle_timeout of both peers." — same page.
- Custom `TransportConfig` is exposed on the `Endpoint::builder` via the per-connection `ConnectOptions` / `QuicTransportConfig`, see [commit 253f4f1](https://github.com/n0-computer/iroh/commit/253f4f1099baac690ea9854f541451a2936eb00d) ("Allow setting a custom quinn::TransportConfig") and [docs.rs/iroh/latest/iroh/endpoint/struct.QuicTransportConfig.html](https://docs.rs/iroh/latest/iroh/endpoint/struct.QuicTransportConfig.html).

**Our equivalent:** `src/p2p/v2/daemon/peer_connect.rs::start_iroh_listener` builds the endpoint with `Endpoint::builder(N0).secret_key(..).alpns(..).hooks(..)` — **no `transport_config(...)` call**. We accept iroh's default: 30 s idle timeout, no keepalive. To stop idle connections dying we run `HealthWorker` Ping/Pong every 30 s.

**Verdict:** **Reinvention of QUIC keepalive at the application layer.** Setting `transport_config.keep_alive_interval = Some(Duration::from_secs(10))` plus `max_idle_timeout = Some(Duration::from_secs(45))` would:
- keep the connection alive automatically (transport-layer PING frames, ~tens of bytes);
- replace `HealthWorker`'s active probing for live connections;
- update `Connection::rtt()` continuously without our code touching it.

The supervisor's `IrohRelayChannel`/`IrohPeerChannel` restart logic should re-evaluate after this — most restarts triggered by "3 missed pings" go away.

**LOC savings:** combined with (A), reduces `health_worker.rs` to a thin offline-peer-tracker; supervisor adapters lose a probe trait impl. **−150 to −250 LOC.**

---

### F. Peer auth / known-peer registry — **ORTHOGONAL (keep our code)**

**iroh upstream:** **Partial overlap.**

- `iroh::discovery::Discovery` trait — node-discovery service for NodeId → relay/direct-addr resolution. — [docs.rs/iroh-net/latest/iroh_net/discovery/trait.Discovery.html](https://docs.rs/iroh-net/latest/iroh_net/discovery/trait.Discovery.html)
- Built-in implementors: `DnsDiscovery`, `mDNS`, `DhtDiscovery` (pkarr), `ConcurrentDiscovery` (combines multiple).
- 1.0 reframes this as `address_lookup::AddressLookup` — [docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html#method.connect](https://docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html).
- NodeId/EndpointId is Ed25519, persisted as the secret key (we already use this via `load_or_create_iroh_secret`).

**Our equivalent:** `src/p2p/v2/peer_registry.rs` — does *much more*: human-readable peer name, alternate-address history, RTT cache, transient-vs-configured distinction, hub-pushed updates, blacklist/health flags, last-sync timestamp.

**Verdict:** **Orthogonal.** iroh's `Discovery` resolves "NodeId → can I connect now?"; our registry persists "what is this peer called, what RTT did I last measure, when did we last sync?" Different layers. Keep.

**Possible refactor (low priority):** our hub-publisher could implement `Discovery` so iroh's `endpoint.connect(node_id)` (no addrs) "just works" via the hub. That would let internal call sites stop building `iroh://<hex>?addrs=...&relay=...` strings and call `endpoint.connect(node_id, ALPN)` directly. **Net −50 to −100 LOC of URL-string plumbing**, no behavior change.

---

### G. Built-in observability — **PARTIAL REINVENTION (medium confidence)**

**iroh upstream:** **Yes — first-class.**

- `Endpoint::metrics() -> &EndpointMetrics` (feature `metrics`, on by default) — "The endpoint internally collects various metrics about its operation." — [docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html](https://docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html) (Source `endpoint.rs#1531-1533`).
- `iroh::metrics::EndpointMetrics` — includes `socket.recv_datagrams`, NetReport counters, etc. — [docs.rs/iroh/latest/iroh/metrics/index.html](https://docs.rs/iroh/latest/iroh/metrics/index.html).
- One-liner OpenMetrics HTTP exporter: `iroh_metrics::service::start_metrics_server` — [docs.rs/iroh-metrics](https://docs.rs/iroh-metrics).
- Registry-based pre-fixing/labelling for multi-component apps.

**Our equivalent:** `src/p2p/v2/transport_health/metrics.rs` — `SupervisorMetrics { channels: Arc<…> }` exposed as JSON at `GET /api/test/transport_health`. Counts: per-channel `last_probe`, `consecutive_failures`, `last_restart`, `last_error`. Tracks our **supervisor's view**, not iroh's transport view.

**Verdict:** **Partial reinvention.** The two metric sets are complementary:
- `SupervisorMetrics` answers "what does our supervisor think?"
- `Endpoint::metrics()` answers "what does iroh think happened on the wire?"

We currently expose only the first. A user diagnosing a stall has to read our log line, then re-derive from there. **The cheap fix**: also expose iroh's metrics through the test-hook endpoint (or as a second endpoint). **No LOC removal — net +30 LOC of glue.**

If we ever ship a real metrics surface (Pro tier?), use `iroh_metrics::registry::Registry` + `register_all(endpoint.metrics())` and avoid duplicating counters that already exist.

---

### H. Router + ProtocolHandler supervision — **JUSTIFIED EXTENSION**

**iroh upstream:** **No auto-restart.**

- `iroh::protocol::Router` + `RouterBuilder::accept(alpn, handler).spawn()` — [docs.rs/iroh/latest/iroh/protocol/struct.Router.html](https://docs.rs/iroh/latest/iroh/protocol/struct.Router.html).
- `Router::shutdown()` propagates handler panics as `JoinError` — [commit fbcaaa5](https://github.com/n0-computer/iroh/commit/fbcaaa56a46b4d2511d65da0330b0cebd89640d1) ("improve Router shutdown").
- `ProtocolHandler::accept` runs on a spawned task; the trait doc explicitly delegates restart-on-failure to the application — [docs.rs/iroh/latest/iroh/protocol/trait.ProtocolHandler.html](https://docs.rs/iroh/latest/iroh/protocol/trait.ProtocolHandler.html).

**Our equivalent:** `src/p2p/v2/transport_health/supervisor.rs` `LivenessSupervisor` watches `ClipshotSyncHandler` indirectly through the `IrohRelayChannel` heartbeat. On panic/wedge, we tear down + rebuild the `Router` (today entangled with the full endpoint rebuild — see §B).

**Verdict:** **Justified extension.** Router does not restart handler tasks on its own; iroh's design assumes the application owns lifecycle. Our `LivenessSupervisor` is the right layer.

Sub-finding: **the `Restartable` impl for Router should be decoupled from the `Endpoint` rebuild** (see §B). A failed `ClipshotSyncHandler` should be rebuilt by constructing a new `Router::builder(endpoint.clone())…spawn()` — the existing `Endpoint` can keep running.

**LOC delta:** zero removal; minor refactor to split "rebuild Router" from "rebind Endpoint".

---

## 3. Recommendations (ranked by impact)

| # | Refactor | Iroh API | Est. LOC delta | Risk |
|---|---|---|---|---|
| **1** | **Set `TransportConfig::keep_alive_interval = ~10s, max_idle_timeout = ~45s`** on the endpoint builder. Remove HealthWorker active probing for connected peers; keep only the "offline peer backoff" projector. | `iroh::endpoint::QuicTransportConfig`, `Builder::transport_config` / `ConnectOptions` ([commit 253f4f1](https://github.com/n0-computer/iroh/commit/253f4f1099baac690ea9854f541451a2936eb00d)) | **−200 LOC** | Low. QUIC keepalive is well-understood; tune interval against typical NAT bind lifetimes (60–120 s). |
| **2** | **Delete `IrohAddrTracker` module; subscribe to `endpoint.watch_addr().stream()` instead.** Hub publisher and registry pruner become two `.stream()` consumers of the same `Watcher`. | `Endpoint::watch_addr() -> Watcher<EndpointAddr>` ([docs.rs/.../endpoint.rs#1201-1215](https://docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html)) | **−350 LOC** (whole `iroh_advertise/` module) | Low. `Watcher` semantics match our broadcast; lagging consumers re-read `.get()` like our docs already describe. |
| **3** | **Replace HealthWorker's "is alive?" projection with `Connection::rtt()` + `Endpoint::remote_info().last_received`.** Keep only adaptive backoff policy (3/10 fails → 5/15 min) as a pure projector over iroh state — no probes spawned. | `Connection::rtt`, `Connection::stats`, `Endpoint::remote_info` ([Connection docs](https://docs.rs/iroh/latest/iroh/endpoint/struct.Connection.html), [RemoteInfo docs](https://docs.rs/iroh/latest/iroh/endpoint/struct.RemoteInfo.html)) | **−150 LOC** (composes with #1) | Low. Behaviour: zero outbound probe traffic for healthy peers. |
| **4** | **Wire `Connection::paths()` into `IrohPeerChannel` so we can detect wedged paths instead of logging `MultipathNotNegotiated`.** Replace the supervisor's per-peer heartbeat with `PathWatcher` state. | `Connection::paths() -> PathWatcher<PathInfoList>` ([Connection docs](https://docs.rs/iroh/latest/iroh/endpoint/struct.Connection.html), [PR #3899](https://github.com/n0-computer/iroh/pull/3899)) | **+80 LOC** (replaces probe code that #3 removes) | Medium. New API; tests against docker-relay E2E recommended. |
| **5** | **Decouple Router rebuild from Endpoint rebuild.** Today `start_iroh_listener` always rebinds the endpoint when the ALPN handler dies. Make the Restartable adapter call `Router::builder(endpoint.clone())…spawn()` against the existing endpoint. | `Router::builder` ([docs.rs/.../protocol/struct.Router.html](https://docs.rs/iroh/latest/iroh/protocol/struct.Router.html)) | **0 LOC removed, ~30 LOC moved** | Medium. Need test that endpoint state survives handler death. |
| **6** | **Audit the `iroh_listener_dead` watchdog against production data.** If no recovery in last 12 months → delete the watchdog and the `start_iroh_listener` rebuild path. Call `Endpoint::network_change()` from OS-level network events instead. | `Endpoint::network_change()` ([endpoint.rs#1564-1570](https://docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html)) | **−80 to −120 LOC** | Medium. Behaviour change. Mitigate with month-long canary. |
| **7** | **Optionally expose `Endpoint::metrics()` through the existing test-hooks HTTP surface** (`/api/test/iroh_metrics` returning OpenMetrics text). Stop duplicating QUIC counters in `SupervisorMetrics`. | `Endpoint::metrics() -> EndpointMetrics` ([endpoint.rs#1531-1533](https://docs.rs/iroh/latest/iroh/endpoint/struct.Endpoint.html)), `iroh_metrics::service` ([docs.rs/iroh-metrics](https://docs.rs/iroh-metrics)) | **+30 LOC** | Low. Diagnostic only. |
| **8** | **Optional: implement `iroh::discovery::Discovery` for our hub publisher** so `endpoint.connect(node_id, ALPN)` resolves through the hub natively. Drop the bespoke `iroh://<hex>?addrs=…&relay=…` string-builder in `daemon/peer_connect.rs`. | `Discovery` trait ([iroh-net docs](https://docs.rs/iroh-net/latest/iroh_net/discovery/trait.Discovery.html)), `Endpoint::Builder::address_lookup` | **−100 LOC** of URL plumbing | Higher; touches every connect call site. |

**Aggregate if #1–#3 applied: roughly −700 LOC of code we built and maintain.** Items #4 and #6 push the total toward **−900 LOC** while simultaneously *improving* observability (path-watcher) and reducing background traffic (no Ping/Pong).

---

## 4. Notes on confidence

- All iroh API claims are against the **current `docs.rs/iroh/latest`** (iroh 1.0.0-rc.0, matching our Cargo.toml). API names like `watch_addr`, `Connection::paths`, `Endpoint::remote_info`, `Connection::rtt/stats`, `network_change`, `metrics`, `Endpoint::online` are all stable in 1.0-rc.
- We are **already** locked to 1.0-rc, so there is no "upgrade first" tax.
- The `ConnectionStats` and `RemoteInfo` field-level docs are slightly thinner on the rendered docs.rs page than the source comments suggest; if exact field semantics matter, cross-check against [`iroh/src/socket/remote_map/remote_state/remote_info.rs`](https://github.com/n0-computer/iroh/blob/main/iroh/src/socket/remote_map.rs).
- The `MultipathNotNegotiated` log we ship is a known *current-state* line from iroh; the upcoming QUIC-MULTIPATH work ([iroh blog](https://iroh.computer/blog/iroh-on-QUIC-multipath)) will make multipath default. Recommendation #4 future-proofs us for that landing.
