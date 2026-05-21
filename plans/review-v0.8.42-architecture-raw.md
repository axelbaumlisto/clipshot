# v0.8.42 Architectural Review — SOLID / DRY / KISS

Scope: clipboard fix series (4afd03b4, 3f4efa60, 0a29b242, e8882361) +
transport-resilience series (e8373e6d → 5f04d712). Verified against the
plan at `plans/transport-resilience-unified.md`.

## 1. Executive Summary

**Verdict**: clipboard series is generally clean, small, policy-isolated.
Transport-resilience series is structurally sound but **does not deliver
the plan's main promise** (net-negative LOC by replacing legacy
machinery). The supervisor sits *alongside* the watchdog + idle_timeout
+ CONNECT_TIMEOUT constants, not in place of them. One adapter
(`IrohPeerChannel`) is fully implemented and unit-tested but **never
wired into the running supervisor**, leaving C1/C2/C5 chaos coverage
unbacked in production.

**Top 3 issues**

1. **BLOCKER — `IrohPeerChannel` is dead code in production**
   (`src/p2p/v2/daemon/initializer.rs:438-462`). The initializer
   registers only `hub-ws` and `iroh-relay`; no `IrohPeerChannel` is
   ever constructed. T5's chaos claim ("C1 + C2 + C5 chaos green") rests
   on an adapter that never runs in the live daemon. ~150 LOC + 7 tests
   exist only to compile.

2. **MAJOR — "delete 600 LOC" promise not kept**. The 120 s tick-stall
   watchdog (`src/p2p/v2/daemon/tick/mod.rs:9-28`),
   `CONNECT_TIMEOUT = 15s` constant (`src/p2p/v2/hub/connection.rs:37`),
   `mark_iroh_listener_dead` callers, and ad-hoc per-tick checks are
   **all still present**. The supervisor is **layered on top**:
   `iroh_listener_dead` now has three writers (watchdog, command_router
   facade, supervisor restart) and one reader. Series diff is
   **+1917 / −65 LOC** in `src/` (verified via
   `git diff --shortstat c89ce316..d17e6d17 -- src/`), the opposite
   sign of the plan's "Definition of Done → net-negative LOC".

3. **MAJOR — `IrohRelayChannel` adds no independent detection**.
   `probe()` reads the same `Arc<AtomicBool>` flag that
   `watchdog_check` (tick/mod.rs:18-20),
   `mark_iroh_listener_dead` (command_facade.rs:62-65), and the
   endpoint-closed check (event_loop.rs:1328) all write to. `restart()`
   sets the **same flag** again. The supervisor is an observation hook
   incrementing `restarts_total`, not a second detection path. C3 would
   have passed with or without it.

---

## 2. SOLID Findings

### 2.1 SRP — clipboard policy split is good
**Note** — `macos_priority.rs` (pure policy on a probed snapshot) +
`detect_pasteboard_presence` (probe-only, lives in `macos.rs`) is a
textbook policy/mechanism split. Same for
`clipboard_paste_policy.rs` (policy) vs
`file_manager::sync_content` (action). Severity: ✅.

### 2.2 SRP/leakage — supervisor restart is co-owned with legacy watchdog
**Severity: Major.**
`IrohRelayChannel::restart()` (`adapters/iroh_relay.rs:97-99`) sets
`iroh_listener_dead = true`. The 120 s tick-stall watchdog
(`tick/mod.rs:12-21`) sets the same flag. `command_facade::mark_iroh_listener_dead`
(`command_facade.rs:62-65`) sets the same flag. Three writers for one
boolean: SRP is held only at the per-call level, not at the system
level. The "single source of truth" promise in
`adapters/iroh_relay.rs` doc-comment is contradicted by the live code.

**Fix**: the plan said delete the watchdog (`tick/mod.rs`
`watchdog_check` block + `WATCHDOG_FORCE_RESTART_SECS`) once T4 lands.
Do it. Either delete the watchdog entirely (~30 LOC + 2 tests) or make
the supervisor *be* the watchdog by probing
`endpoint.is_closed()` directly instead of polling the flag.

### 2.3 LSP — `Restartable` contract too permissive in practice
**Severity: Minor.**
All three adapters' `restart()` impls always return `Ok(())`:
- `hub_ws.rs:96-101`: notify_one then Ok
- `iroh_relay.rs:97-99`: atomic store then Ok
- `iroh_peer.rs:99-104`: HealthState::reset then Ok

The `Result<(), RestartError>` return type and the supervisor's
"failed restart will be retried" comment (`supervisor.rs:148-150`)
support a contract no implementation ever exercises. Either some impl
should actually fail synchronously (e.g. surface a timeout) or the
return type should be `()` and the supervisor's retry logic simplified.

### 2.4 ISP — `Heartbeat` + `Restartable` are right-sized
**Note** — two methods, no fat trait. ✅

### 2.5 DIP — supervisor depends on traits; adapters depend on concrete
**Note**: `HubWsChannel` depends on the concrete `HubClientHealthHandle`
(`adapters/hub_ws.rs:21`); `IrohRelayChannel` depends on
`Arc<AtomicBool>`; `IrohPeerChannel` depends on the concrete
`HealthState`. This is fine — adapters are by definition the
concrete-to-abstract bridge. ✅

### 2.6 OCP — new transport adds an adapter, no edits to supervisor
**Note** — verified at `adapters/mod.rs`: `IrohRelayChannel` and
`IrohPeerChannel` were added behind `cfg(feature = "iroh")` without
touching `supervisor.rs` or `mod.rs`. ✅

---

## 3. DRY Findings

### 3.1 Two `ContentTypeKind`-shaped enums in the codebase
**Severity: Minor.**
- `src/clipboard/macos_priority.rs:43-52`: `PasteboardSource { Files, Image, Text, Empty }`
- `src/sync/clipboard_paste_policy.rs:49-54`: `ContentTypeKind { Text, Image, File }`

Different domains (probe vs sync), so not strict duplicates. But the
already-existing `crate::content::ContentType` (Text/Image/File) is the
canonical one, and `ContentTypeKind` exists only to drop the
mime/name payload from the original enum. Either:
- introduce `ContentType::kind() -> ContentKind` once in
  `src/content.rs` and let both modules consume it, or
- accept the duplication as policy-isolation (current state).

Net: ~20 LOC saved if consolidated, but the modules genuinely have
different vocab ("Files" vs "File" reflects the macOS pasteboard model
of an array of files). Keep separate — this is a Nit at most.

### 3.2 `HubClient::last_recv` + `HubClientHealthHandle::last_recv`
**Severity: Minor.**
`HubClient::last_recv()` (`hub/connection.rs:253-255`) and
`HubClient::request_reconnect()` (`hub/connection.rs:266-268`) are
public methods that no production caller uses. `grep` confirms the only
external callers are the supervisor (via `HubClientHealthHandle`) and
one test (`connection.rs:956`). The handle's two methods (`hub_ws.rs:80`,
`hub_ws.rs:100`) do the same thing through a copy of the same Arcs.

**Fix**: delete `HubClient::last_recv` and `HubClient::request_reconnect`
(production-unused), keep only the handle. ~20 LOC net.

### 3.3 Three writers, one `iroh_listener_dead` flag
**Severity: Major.** See 2.2. Same flag set by:
- `tick/mod.rs:18-20` (watchdog @ 120s)
- `daemon/command_facade.rs:62-65` (mark_iroh_listener_dead)
- `daemon/event_loop.rs:1328, 1475, 1538` (multiple reactive paths)
- `transport_health/adapters/iroh_relay.rs:98` (supervisor restart)

This *is* DRY at the flag-storage layer (single Atomic) but represents
**four uncoordinated detection paths feeding one bit**. The plan said
the supervisor would *replace* these. It didn't.

### 3.4 `HealthState::backoff_for` vs `Policy.consecutive_failures_before_restart`
**Severity: Minor / not a real overlap.**
- `HealthState::backoff_for(failures, base)` → maps a failure count to
  a *duration* before the next probe.
- `Policy.consecutive_failures_before_restart` → number of consecutive
  failures the supervisor tolerates before restart.

Complementary, not overlapping. But the practical effect of
`IrohPeerChannel.escalate_at_failures = BACKOFF_FAILS_LONG (10)`
(`adapters/iroh_peer.rs:43, 57`) means the supervisor stays silent for
the first 10 failures (≈ 5–15 minutes via HealthState's own backoff).
So the supervisor's claimed "30s detection window from
`Policy::default()`" does **not** apply to per-peer channels — they
take 5–15 minutes to surface. Plan vs reality drift. Document it or
lower `escalate_at_failures` to e.g. `BACKOFF_FAILS_MEDIUM = 3` to
match the global Policy. Currently moot anyway: see 1 (the adapter is
never wired).

---

## 4. KISS Findings

### 4.1 `SupervisedChannel` requires two separate `Arc<dyn …>` for one struct
**Severity: Nit.**
`supervisor.rs:14-31` takes `Arc<dyn Heartbeat>` + `Arc<dyn Restartable>`.
All three adapters implement both on the same struct, so wiring in the
initializer is the verbose dance at `initializer.rs:441-442`,
`456-458`, and in three test files:

```rust
let hub_ch: Arc<HubWsChannel> = Arc::new(HubWsChannel::new(...));
let h: Arc<dyn Heartbeat>   = hub_ch.clone();
let r: Arc<dyn Restartable> = hub_ch;
channels.push(SupervisedChannel::new(h, r, Policy::default()));
```

Two Arc clones for one object × N channels. The plan rejected
"`Heartbeat: Probe`" generics, but a single
`trait HeartbeatAndRestartable: Heartbeat + Restartable {}` with a
blanket impl, plus `SupervisedChannel::single(Arc<dyn _>)` constructor,
would halve the cloning ceremony with no loss of flexibility.
~6 LOC saved per supervisor wire-up.

### 4.2 `HubClientHealthHandle` is a wrapper that adds nothing
**Severity: Minor.**
`hub/connection.rs:117-132` defines a struct holding the same two Arcs
that `HubClient` already holds, exposing the same two methods. The
doc-comment justifies it as "avoiding wrapping HubClient in Arc". But:
- HubClient is in `Option<HubClient>` in the daemon today; wrapping in
  Arc is one ownership change, not a force.
- The handle is **38 LOC + Clone derive + 2 method impls** to avoid
  one Arc.
- It also forces the duplicate methods flagged in 3.2.

**Fix**: drop the handle struct; supervisor takes `Arc<HubClient>`
directly (or accept a small refactor: store `Arc<HubClient>` in the
daemon). Net delete ≈ 30 LOC.

### 4.3 `Policy.tick_interval` is per-channel but supervisor takes the min
**Severity: Nit.**
`supervisor.rs:76-79` computes `min` of all tick intervals. In practice
every channel uses `Policy::default()`. Either honour per-channel
intervals (separate `tokio::interval`s in a `select!`) or remove
per-channel `tick_interval` and make it a supervisor-level field.
Saves ~4 LOC + clarifies the contract.

### 4.4 `macos_priority::PasteboardPresence` is appropriate
**Note** — three booleans + 4-variant enum is the smallest possible
faithful representation of "what NSPasteboard reports". Not
overengineered. ✅

### 4.5 `Heartbeat::id()` returns `String` (allocates per drive)
**Severity: Nit.**
Plan draft used `&'static str`; implementation moved to `String`
because `IrohPeerChannel` builds an id-suffix per peer
(`adapters/iroh_peer.rs:49-55`). Result: every `drive_channel` call
clones a `String` for metrics keying (`supervisor.rs:106`). One short
allocation per channel per tick — cheap, but the plan's KISS posture
suggests `Cow<'static, str>` or caching the `String` (already done on
the struct as a field, so the per-call `clone()` is the only waste).

---

## 5. Hidden Coupling / Correctness Concerns

### 5.1 BLOCKER — `IrohPeerChannel` never wired
**Severity: Blocker for the plan's chaos coverage claim.**
`initializer.rs:438-462` registers exactly two channels: `hub-ws` and
`iroh-relay`. There is no loop over `peer_registry` peers, no
`IrohPeerChannel::new(...)` call anywhere in `src/p2p/v2/daemon/`. The
adapter (`adapters/iroh_peer.rs`, 217 LOC including tests) compiles, has
7 unit tests, but provides zero production value: chaos scenarios C1
(udp-blackhole-peer) and C5 (multipath-confusion) rely on per-peer
restart, which never fires.

**Fix**: in `initializer.rs`, after building peer registry, iterate
configured peers, build an `IrohPeerChannel` per address keyed by
`peer_addr`, add to `channels`. The list must be **dynamic** (peers
join/leave at runtime) — likely needs a "subscribe to peer registry"
hook on the supervisor or a re-spawn on peer-list change. This is the
real T5 work that was skipped.

### 5.2 `set_image_with_path_text` default impl silently loses image on Linux/Windows
**Severity: Major.**
`clipboard/mod.rs:178-187`:
```rust
fn set_image_with_path_text(&self, _image_data: &[u8], path_text: &str) -> ... {
    // Default fallback: text only. macOS overrides for true
    // multi-type write.
    self.set_content(&ClipContent::text(path_text))
}
```

On a Linux desktop daemon (X11/Wayland with `has_display() == true`),
`sync_content` for an Image will now overwrite the user's clipboard
with the path text, **dropping the image bytes the user just copied**.
That's a behavioural regression introduced by e8882361 for non-macOS
desktop users.

The justification comment ("paste-as-image consumers are rare") is
factually wrong on Linux — Slack/Discord/Telegram desktop all consume
image clipboards routinely.

**Fix options**:
- Make the default impl write the image only and leave text alone (no
  regression, but loses the path-paste benefit on Linux).
- Make the default impl a no-op and gate the policy: only return
  `true` from `should_overwrite_clipboard_with_sync_path` for Image
  content when the platform's clipboard genuinely supports multi-type
  writes. The policy already has `has_display`; add a
  `supports_multi_type_write` capability boolean.
- Implement true multi-type write on `X11Clipboard` / `WaylandClipboard`
  (xclip can take MIME types; wl-copy has `--type`).

### 5.3 `suppress_next_hash` is a single-shot global slot
**Severity: Minor (current callers safe, future-coupling risk).**
`clipboard_cache.rs:34-79`: one `OnceLock<Mutex<Option<(u64, Instant)>>>`
global. If two writes are queued (e.g. paste_path + file sync) the
second `suppress_next_hash(...)` overwrites the first. Today the call
sites are non-concurrent:
- `gui/commands.rs:727` (paste_path, user-driven, rare)
- `sync/file_manager.rs:225` (per file sync, serial)

But there is no mutual exclusion and no test for the "concurrent
suppress" case. Hash collision on `u64` is astronomically unlikely; the
realistic risk is **lost suppression** when two writes happen between
clipboard polls.

Note also the policy comment in `clipboard_paste_policy.rs:147`:
> Cmd+B (paste_path_hotkey) still toggles back to the original file URL

This implicitly assumes the cache survives the new path write. Verified
in `sync/file_manager.rs:225` — only `suppress_next_hash` is called,
not `cache::store_with_path`. So Cmd+B will **not** find the original
file URL after the new policy fires; the toggle will fail. There is no
test for this interaction.

### 5.4 `get_settings_cache()` reads disk on the file-sync hot path
**Severity: Minor (perf + naming).**
`p2p/control/settings.rs:563-565`: despite the name, this calls
`load_app_settings()` which parses the TOML file. Called from
`file_manager.rs:212` once per `sync_content` call (i.e. every file
that gets synced). For a large incoming sync (50 files in a tar-like
burst) that's 50 disk reads where one would do.

**Fix**: rename to `load_settings_from_disk()` or actually implement
the in-memory cache the name promises. The latter would also fix
test flakiness against process-wide settings state.

### 5.5 `has_display()` reads env vars (Linux)
**Severity: Minor (testability hazard).**
`display_utils.rs:6-29` reads `DISPLAY` / `WAYLAND_DISPLAY` globals.
`file_manager.rs:197` calls it on every sync. The test
`test_sync_updates_clipboard_with_path` at `file_manager.rs:651` had to
weaken its assertion to `count <= 1` *because* the test cannot
deterministically control `has_display` cross-platform. That is the
test admitting the abstraction's flaw.

**Fix**: inject `has_display` (boolean) into `FileSyncManager` at
construction. The struct already takes `enabled: bool`; one more bool
is cheaper than the current global-env coupling. Net: the test becomes
strict, and behaviour is documented.

### 5.6 Supervisor `JoinHandle` is stored but never aborted
**Severity: Minor.**
`initializer.rs:474-475` stores `daemon.transport_health_task`. No
`Drop` impl, no shutdown call. On hub reconfigure (settings change),
the daemon currently builds a new HubClient and would spawn a second
supervisor while the first leaks. Not yet exercised because hub URL
change is rare, but a real leak.

### 5.7 `pick_source` is unused for the image fallback path
**Severity: Nit.**
`macos.rs::get_content` (line ~280): if `pick_source` returns
`PasteboardSource::Image`, the code falls through to the existing
"Image probe (also the fallback when files probe failed)" block. So
the `Image` and `Empty` variants of `PasteboardSource` are
unused — the `match` only differentiates `Files` from "everything
else, fall through". Could be `if presence.has_files { … } else {
fall-through }` and skip the enum entirely. Mild over-modelling.

---

## 6. Test Gaps

1. **No production wiring test for `IrohPeerChannel`** (cross-ref 5.1).
   The adapter's 7 tests pass; no test asserts it is registered with
   the supervisor when the daemon initialises. A minimal test:
   spawn daemon, query `/api/test/transport_health`, assert at least
   one `iroh-peer-*` key when ≥ 1 peer is configured.

2. **No `set_image_with_path_text` contract test on Mac**
   (cross-ref 5.2). `macos.rs` has `test_macos_new`, `test_default`,
   threading tests, but no test that asserts both PNG bytes **and**
   UTF-8 text are present on the pasteboard after this method.
   The trait default fallback (`mod.rs:178`) has no test asserting it
   *fails fast* on Linux/Windows Image content (which is the
   behavioural regression in 5.2).

3. **No interaction test for `suppress_next_hash` + `sync_content`**
   (cross-ref 5.3). The gate is unit-tested
   (`clipboard_cache.rs:209-243`) but the integration ("daemon writes
   path → next tick observes the hash → does NOT broadcast") is only
   covered by manual verification per the commit message of 0a29b242.

4. **Watchdog + supervisor cohabitation untested**. With
   `iroh_listener_dead` now having three writers, no test exercises the
   sequence: watchdog flips → supervisor probe sees flag → supervisor
   restart sets flag again → tick rebuild clears flag → next probe
   passes. Could happen in CI in 20 lines.

5. **Test claim vs reality for `test_sync_updates_clipboard_with_path`**
   (`file_manager.rs:651`). Asserting `count <= 1` is **not a
   regression test** — the test passes whether the feature works or
   not. Cross-ref 5.5: inject `has_display`, then this test can
   strictly check that `count == 1` on `has_display = true`.

6. **TDD claim, RED-first**
   - `macos_priority.rs` tests: clearly RED-first possible (pure
     function). ✅
   - `clipboard_paste_policy.rs` tests: RED-first verified by reading
     the policy — without the function body, all 6 fail. ✅
   - `transport_health/policy.rs::default_policy_matches_v0_8_24_tuning`
     is a snapshot test of constants; it doesn't drive new behaviour.
     OK as a regression-prevention pin.
   - `iroh_peer.rs` tests use real `HealthState` not a mock — fine for
     in-process state, but calling them "unit tests" is generous;
     they're tight integration tests against `HealthState`.

---

## 7. Refactor Candidates (ordered by impact, smallest first)

1. **Delete `HubClient::last_recv` + `HubClient::request_reconnect`**
   (cross-ref 3.2). Scope: 4 LOC removed in `connection.rs`. No
   dependencies. ~−20 LOC including doc-comments. Safe.

2. **Collapse `set_image_with_path_text` default impl risk**
   (cross-ref 5.2). Scope: change `clipboard_paste_policy::should_overwrite`
   to require an explicit `supports_multi_type_write: bool` and wire
   it from the platform clipboard. ~+15 LOC policy, ~−5 LOC default
   trait method, **0 user-visible regressions on Linux/Windows**.

3. **Inject `has_display` into `FileSyncManager`** (cross-ref 5.5).
   Scope: 1 field, 1 ctor arg threaded through ~4 call sites. ~+15 LOC.
   Net win: strict test, no global env coupling. Worth it.

4. **Delete `HubClientHealthHandle`, take `Arc<HubClient>` directly**
   (cross-ref 4.2). Scope: ~−30 LOC in connection.rs; ~−10 LOC in
   `adapters/hub_ws.rs`. Requires daemon to hold `Arc<HubClient>`
   instead of `Option<HubClient>` (~5 LOC). Net: ~−35 LOC.

5. **Honour the plan: delete the legacy watchdog + idle_timeout
   constants** (cross-ref 1, 2.2, 3.3). Scope:
   - delete `tick/mod.rs` watchdog block (≈ 30 LOC + 4 tests in
     `event_loop.rs::tests` that assert `iroh_listener_dead` is set at
     120s)
   - delete `CONNECT_TIMEOUT` constant + uses (`hub/connection.rs:37,
     338, 348`) → use `Policy.probe_timeout` ≈ 15 LOC
   - move `command_facade::mark_iroh_listener_dead` callers to call
     `IrohRelayChannel::restart` via metrics handle, or accept that
     mark-dead is a fast path the supervisor doesn't need to replicate
   Net target: ~−150 LOC matching the plan's intent. Requires the
   supervisor's iroh adapter to probe `endpoint.is_closed()` directly
   instead of reading a flag.

6. **Wire `IrohPeerChannel` into the supervisor** (cross-ref 5.1).
   Scope: ~+30 LOC in `initializer.rs` to iterate peers, plus a hook
   for peer-list-change to re-register (likely in
   `peer_registry.rs`). Net +50 LOC, **the actually-promised T5 work**.
   Without this, the entire IrohPeerChannel file (≈ 220 LOC) and 7
   tests are dead weight and should be feature-gated to `cfg(test)` if
   wiring is deferred to a later release.

7. **Optional: introduce `trait HeartbeatAndRestartable` blanket impl**
   (cross-ref 4.1). Scope: ~+15 LOC trait + blanket, ~−4 LOC per
   call site × 3 sites = ~−12 LOC. Net ~+3 LOC. **Skip unless wiring
   IrohPeerChannel adds enough sites to justify the trait** (it
   probably will: per-peer registration multiplies the ceremony).

---

## Appendix — Net LOC verification

- Transport-resilience series (`c89ce316..d17e6d17`):
  `git diff --shortstat -- src/` → **+1917 / −65** lines.
- Clipboard series (`5e9f7944~1..e8882361`):
  → **+768 / −34** lines (mostly tests + new policy modules; acceptable
  for the feature, but the deletion target was not part of the plan
  for this series).
- Plan's "Definition of Done" required:
  `git diff --shortstat v0.8.24..v0.9.0` to be net-negative. It is not.

The supervisor is a sound abstraction; it is just not the *replacement*
the plan said it would be. Treat the current state as T2–T6 landed,
**T7-deletion-and-rewire deferred**.
