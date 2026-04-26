---
layout: default
title: Release Checklist
---

# Release Checklist

## 1) PR reviewer required checks (blocking)

Run locally (or verify in CI) before merge to `main`:

| Check | Command | Required |
|---|---|---|
| Lint | `cargo clippy --all-targets -- -D warnings` | Yes |
| Complexity budget (critical strict mode) | `STRICT_FUNCTION_BUDGET=true STRICT_FUNCTION_BUDGET_MODE=critical python3 scripts/ci/check-complexity-budget.py` | Yes |
| TDD coverage gate | `bash scripts/ci/check-tdd-coverage.sh` | Yes |
| Rust lib tests (deterministic proptest seed) | `PROPTEST_RNG_ALGORITHM=cc PROPTEST_RNG_SEED=20260405 PROPTEST_DISABLE_FAILURE_PERSISTENCE=true cargo test --lib` | Yes |
| Rust bin tests | `cargo test --bin clipshot -q` | Yes |
| Web smoke | `cd web && bun install --frozen-lockfile && bun x playwright test --grep @smoke --project=chromium` | Yes |

Convenience wrapper:

```bash
bash scripts/ci/run-required.sh
```

## 2) Nightly / non-blocking quality lanes

| Lane | Command | Blocking |
|---|---|---|
| macOS hotkey+smoke | `make ci-macos-hotkey-smoke` | No (nightly/manual) |
| Mutation pilot (sync_status) | `make ci-mutation-pilot` | No (report-only default) |
| Mutation pilot (sync_transition) | `make ci-mutation-sync-transition` | No (report-only default) |
| Mutation pilot (uri) | `make ci-mutation-uri` | No (report-only default) |
| Docker Pro | `make ci-docker-pro` | No |
| Docker Relay | `make ci-docker-relay` | No |
| Iroh stress (opt-in) | `RUN_IROH_STRESS=true make ci-nightly` | No |

Nightly runner:

```bash
bash scripts/ci/run-nightly.sh
```

## 3) Release-specific notes

- Ensure macOS lane (`ci-macos-hotkey-smoke`) is green for desktop/hotkey-impacting changes.
- If mutation score is below threshold, log follow-up issue even in report-only mode.
- Keep daemon boundary invariant: wrappers remain thin (`parse/validate/delegate/respond`).

---

## 4) Multi-agent ownership compliance checklist

Applies to every PR that was developed under a wave-based multi-agent plan.  
Contract: `docs/superpowers/specs/2026-04-03-multi-agent-execution-contract.md`  
Ownership board: `docs/superpowers/specs/2026-04-06-wave-ownership-board.md`

### 4.1 Per-PR verification (reviewer must confirm all)

- [ ] **File-family ownership respected** — every file changed in this PR belongs to  
  the file-family assigned to this agent/task in the wave ownership board (§1 table).  
  Run: `git diff --name-only origin/main | sort` and compare against assigned family.

- [ ] **No-overlap rule satisfied** — no file changed in this PR is also changed by a  
  concurrent agent in the same wave. Verify using the No-Overlap Matrix (§2 of the  
  ownership board).

- [ ] **Handoff record complete** — the agent's handoff entry in the ownership board  
  Handoff Log (§8) includes: branch/commit, owned files table, verification commands  
  run, pass/fail summary, and residual risks. No required field is blank.

- [ ] **No business logic in wrappers** — GUI (`src/gui/commands.rs`,  
  `src/gui/daemon_bridge.rs`), HTTP (`src/http/handlers*.rs`), and CLI wrappers  
  contain only parse/validate/delegate/respond logic. Verify with architecture  
  boundary tests:
  ```bash
  cargo test --test daemon_boundary_spec
  ```
  All 5 tests must pass.

- [ ] **Daemon remains single source of truth** — no peer management, gossip, or sync  
  logic was added outside `src/p2p/v2/daemon/`. Grep check:
  ```bash
  git diff origin/main -- src/gui/ src/http/ | grep -E '(PeerManager|AsyncPeer|gossip|mesh\.rs)' \
    | grep -v 'use\|mod\|//'
  ```
  Output must be empty.

- [ ] **Complexity allowlist not expanded without approval** — `scripts/ci/check-complexity-budget.py`  
  allowlist entries may only increase if the PR description includes: rationale,  
  split plan, and a follow-up task reference. Check:
  ```bash
  git diff origin/main -- scripts/ci/check-complexity-budget.py | grep 'ALLOWLIST'
  ```

- [ ] **Wave gate green before next wave** — if this PR is the last task in a wave,  
  the integration gate must pass before any Wave N+1 PR is merged:
  ```bash
  cargo clippy --all-targets -- -D warnings
  cargo test --lib --tests
  cargo test --doc
  ./tests/e2e/scripts/ci/run-relay.sh
  ./tests/e2e/scripts/ci/run-relay-soak.sh 3
  ```

### 4.2 Escalation check

If the PR author needed to touch a file outside their assigned file-family:

- [ ] An escalation note was filed in the ownership board (§7 template) **before**  
  any out-of-scope file was committed.
- [ ] The escalation was resolved (task moved, re-sliced, or reassigned) and the  
  wave assignment table (§1) and no-overlap matrix (§2) were updated accordingly.
- [ ] No out-of-scope commit exists that bypasses an unresolved escalation.

### 4.3 Final release candidate gate (after Wave 5 / all waves complete)

Run in addition to §1 required checks:

```bash
# Architecture boundary
cargo test --test daemon_boundary_spec

# Complexity (normal mode snapshot)
python3 scripts/ci/check-complexity-budget.py

# Property tests
cargo test --lib uri_prop_tests
cargo test --lib sync_status_prop_tests

# Mutation pilots (all three profiles)
make ci-mutation-pilot
MUTATION_PROFILE=sync_transition ./scripts/ci/run-mutation-pilot.sh
MUTATION_PROFILE=uri ./scripts/ci/run-mutation-pilot.sh

# Relay E2E + soak ×5 (stricter than per-wave ×3)
./tests/e2e/scripts/ci/run-relay.sh
./tests/e2e/scripts/ci/run-relay-soak.sh 5
```

All commands must exit 0. Relay soak must report `pass=5 fail=0 cycles=5`.  
Mutation scores must meet or exceed: `sync_status` ≥ 80%, `sync_transition` ≥ 80%, `uri` ≥ 70%.

### 4.4 Scorecard reference

For before/after quality metrics and evidence links covering the full  
30-day hardening run, see:  
`docs/superpowers/specs/2026-05-06-quality-final-scorecard.md`
