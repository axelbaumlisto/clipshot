---
layout: default
title: E2E Testing
---

# E2E Testing Guide

## Overview

Clipshot GUI uses **Playwright** for end-to-end testing. Tests follow the **Page Object Model** pattern and adhere to **SOLID**, **DRY**, **KISS**, and **TDD** principles.

## Quick Start

```bash
# Navigate to web directory
cd web

# Install dependencies (if not already installed)
npm install

# Run all E2E tests
npm run test:e2e

# Run specific test file
npx playwright test 02-peers.spec.ts

# Run tests in UI mode (interactive)
npm run test:e2e:ui

# Debug a specific test
npm run test:e2e:debug

# Run tests in headed mode (see browser)
npm run test:e2e:headed

# View test report
npm run test:e2e:report
```

## CI Tiers (GitVerse Actions)

```bash
# Required gate
make ci-required

# Manual/nightly heavy suites
make ci-nightly

# Optional policy check
make ci-flaky-check

# Optional iroh stress lane (manual/nightly)
make ci-iroh-stress
```

Notes:
- `ci-nightly` requires `clipshot_portal` repo available at `../clipshot_portal` (or set `PORTAL_SRC=/path/to/clipshot_portal`).
- In `Docker E2E` workflow heavy job, enable input `run_iroh_stress=true` to run ignored iroh stress tests.
- Docker logs are collected via `tests/e2e/scripts/ci/collect-logs.sh` for CI artifacts on failures.

## Relay Reliability + Soak

```bash
# Single relay lane run
PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-relay.sh

# Soak (N cycles), prints SOAK_RESULT pass=X fail=Y
PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-relay-soak.sh 5
```

- `run-relay.sh` enables strict peer-discovery assertions only when stack health precheck passes (`RELAY_STRICT_DISCOVERY=1`).
- This avoids false-reds in degraded environments while enforcing stronger assertions on healthy stacks.
- On failure, `run-relay.sh` writes diagnostics to `tests/e2e/logs/relay-fail-<ts>/` (compose ps/logs + HTTP snapshots + container logs).
- `run-relay-soak.sh` writes per-cycle logs and summary to `tests/e2e/logs/relay-soak-<ts>/`.

Useful env vars:
- `RELAY_DIAG_DIR=/custom/path`
- `RELAY_DUMP_DIAGNOSTICS_ON_FAIL=0`
- `RELAY_KEEP_STACK_ON_FAIL=1`
- `SOAK_OUT_ROOT=/custom/log/root`

## Self-Contained Docker E2E Launcher

```bash
# Dry-run (prints selected suites + summary line format)
./tests/e2e/scripts/ci/run-e2e-self-contained.sh --dry-run --all

# Relay only
PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-e2e-self-contained.sh --relay

# Pro + Relay (default if no suite flags)
PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-e2e-self-contained.sh

# All suites (relay + pro + p2p)
PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-e2e-self-contained.sh --all

# macOS hybrid (local GUI + docker suites)
PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-macos-gui-docker.sh

# visual policy
# auto (default): visual if available, fallback to nonvisual
# MACOS_DOCK_MODE=auto PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-macos-gui-docker.sh

# require visual checks (fail if screen capture/TCC unavailable)
# MACOS_DOCK_MODE=visual PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-macos-gui-docker.sh

# force nonvisual mode
# MACOS_DOCK_MODE=nonvisual PORTAL_SRC=/path/to/clipshot_portal ./tests/e2e/scripts/ci/run-macos-gui-docker.sh
```

SLA env knobs for macOS lane:
- `MACOS_VISUAL_THRESHOLD_MS`, `MACOS_VISUAL_P95_SLA_MS`
- `MACOS_STATUS_THRESHOLD_MS`, `MACOS_STATUS_P95_SLA_MS`

Summary format:
```text
E2E_SELF_CONTAINED_RESULT pass=<N> fail=<M> suites=<csv>
```

Smoke check for summary contract:
```bash
./tests/e2e/scripts/ci/test-run-e2e-self-contained-smoke.sh
```

## Multi-Agent Ownership + Wave Sync

For branch-wide parallel execution rules, see:
`docs/superpowers/specs/2026-04-03-multi-agent-execution-contract.md`

Minimum integration gate after each wave:
- `cargo clippy --all-targets -- -D warnings`
- `cargo test --lib --tests`
- `cargo test --doc`
- `./tests/e2e/scripts/ci/run-relay.sh`

## Test Architecture

### Page Object Model (POM)

Tests use the Page Object Model to encapsulate page interactions:

```typescript
// Page Object
export class PeersPage extends BasePage {
  private get addPeerButton() {
    return this.getByRole('button', { name: /add peer/i });
  }

  async addPeer(name: string, address: string) {
    await this.openAddPeerDialog();
    await this.nameInput.fill(name);
    await this.addressInput.fill(address);
    await this.submitButton.click();
  }

  async expectPeerExists(peerName: string) {
    await expect(this.getPeerCard(peerName)).toBeVisible();
  }
}

// Test
test('should add peer', async ({ peersPage }) => {
  await peersPage.navigate();
  await peersPage.addPeer('Test', '192.168.1.1:9999');
  await peersPage.expectPeerExists('Test');
});
```

### SOLID Principles

1. **Single Responsibility**: Each page object handles one page
2. **Open/Closed**: Extend BasePage without modifying it
3. **Liskov Substitution**: All page objects can be used as BasePage
4. **Interface Segregation**: Focused page interfaces
5. **Dependency Inversion**: Tests depend on Page Objects, not DOM

### DRY (Don't Repeat Yourself)

- **Reusable Page Objects**: Common actions in page classes
- **Shared Fixtures**: Custom fixtures for page injection
- **Test Data**: Centralized in `tests/e2e/data/testData.ts`
- **Base Page**: Common operations in `BasePage.ts`

### KISS (Keep It Simple)

- Clear test names matching scenarios
- Minimal test setup (fixtures handle it)
- Readable assertions
- No over-abstraction

### TDD (Test-Driven Development)

Our workflow:
1. Write failing test
2. Add `data-testid` to component
3. Test passes
4. Refactor if needed

## Project Structure

```
web/
├── tests/
│   └── e2e/
│       ├── fixtures/
│       │   └── index.ts          # Custom Playwright fixtures
│       ├── pages/
│       │   ├── BasePage.ts       # Base page class
│       │   ├── OverviewPage.ts   # Overview page object
│       │   ├── PeersPage.ts      # Peers page object
│       │   ├── ActivityPage.ts   # Activity page object
│       │   └── SettingsPage.ts   # Settings page object
│       ├── data/
│       │   └── testData.ts       # Test data constants
│       ├── 01-overview.spec.ts   # Overview tests (8 scenarios)
│       ├── 02-peers.spec.ts      # Peers tests (14 scenarios)
│       ├── 03-settings.spec.ts   # Settings tests (16 scenarios)
│       ├── 04-activity.spec.ts   # Activity tests (12 scenarios)
│       ├── 05-navigation.spec.ts # Navigation tests (8 scenarios)
│       └── 06-integration.spec.ts # Integration tests (8 scenarios)
├── playwright.config.ts          # Playwright configuration
└── package.json                  # Scripts
```

## Writing Tests

### 1. Create Page Object

```typescript
// tests/e2e/pages/NewPage.ts
import { BasePage } from './BasePage';
import { expect } from '@playwright/test';

export class NewPage extends BasePage {
  // Locators (private)
  private get myButton() {
    return this.getByRole('button', { name: /my button/i });
  }

  // Actions (public)
  async clickMyButton() {
    await this.myButton.click();
  }

  // Assertions (public)
  async expectButtonVisible() {
    await expect(this.myButton).toBeVisible();
  }
}
```

### 2. Add to Fixtures

```typescript
// tests/e2e/fixtures/index.ts
import { NewPage } from '../pages/NewPage';

type Fixtures = {
  // ... existing fixtures
  newPage: NewPage;
};

export const test = base.extend<Fixtures>({
  // ... existing fixtures
  newPage: async ({ page }, use) => {
    await use(new NewPage(page));
  },
});
```

### 3. Write Test

```typescript
// tests/e2e/07-new-feature.spec.ts
import { test, expect } from './fixtures';

test.describe('New Feature', () => {
  test('TC-XXX: should do something', async ({ newPage }) => {
    await newPage.navigate();
    await newPage.clickMyButton();
    await newPage.expectButtonVisible();
  });
});
```

### 4. Add data-testid to Component

```tsx
// src/components/MyComponent.tsx
export function MyComponent() {
  return (
    <div data-testid="my-component">
      <button data-testid="my-button">Click me</button>
    </div>
  );
}
```

## Test Data

Centralized test data in `tests/e2e/data/testData.ts`:

```typescript
export const TestPeers = {
  validPeer: {
    name: 'Test Node 1',
    address: '192.168.1.100:9999',
  },
};

// Usage in tests
import { TestPeers } from './data/testData';

test('should add peer', async ({ peersPage }) => {
  await peersPage.addPeer(TestPeers.validPeer.name, TestPeers.validPeer.address);
});
```

## Best Practices

### Selectors Priority

1. **Role-based** (preferred): `getByRole('button', { name: /add/i })`
2. **Test IDs**: `getByTestId('peer-card')`
3. **Text**: `getByText('No peers')`
4. **CSS**: Only as last resort

### Waiting Strategies

```typescript
// ✅ Good - use waitForReady()
async navigate() {
  await this.goto('/peers');
  await this.waitForReady();
}

// ✅ Good - implicit waiting with assertions
await expect(this.peerCard).toBeVisible();

// ❌ Bad - arbitrary timeouts
await this.page.waitForTimeout(5000);
```

### Assertions

```typescript
// ✅ Good - specific assertions
await expect(this.peerCards).toHaveCount(2);
await expect(this.statusBadge).toContainText('Connected');

// ❌ Bad - generic assertions
expect(await this.peerCards.count()).toBe(2);
```

## CI/CD Integration

Tests run automatically on:
- Push to `main` or `develop` branches
- Pull requests to `main`
- Manual workflow dispatch

See `.github/workflows/e2e-tests.yml` for CI configuration.

### GitHub Actions

```yaml
# View workflow runs
https://github.com/<org>/<repo>/actions

# Artifacts include:
- playwright-report/ (HTML report)
- test-results/ (screenshots, videos, traces)
```

## Debugging

### Local Debugging

```bash
# Debug mode (step through tests)
npm run test:e2e:debug

# UI mode (interactive)
npm run test:e2e:ui

# Headed mode (see browser)
npm run test:e2e:headed
```

### CI Debugging

1. Check GitHub Actions workflow logs
2. Download `playwright-report` artifact
3. Open `index.html` locally to view report
4. Check screenshots/videos in `test-results/`

### Common Issues

**Issue**: Test fails with "element not found"
**Fix**: Check `data-testid` attribute exists, use `waitForReady()`

**Issue**: Flaky tests
**Fix**: Avoid `waitForTimeout()`, use proper assertions

**Issue**: Tauri dev server not starting
**Fix**: Check `playwright.config.ts` webServer config, increase timeout

## Test Coverage

Current coverage: **66 test scenarios** across 6 files

- ✅ Overview (8 tests)
- ✅ Peers (14 tests)
- ✅ Settings (16 tests)
- ✅ Activity (12 tests)
- ✅ Navigation (8 tests)
- ✅ Integration (8 tests)

See [TEST_SCENARIOS.md](./TEST_SCENARIOS.md) for full scenario list.

## Performance

- **Parallel execution**: Tests run in parallel (up to 4 workers)
- **Fast feedback**: ~30-60 seconds for full suite
- **Caching**: WebServer reuse in local development

## Resources

- [Playwright Documentation](https://playwright.dev/)
- [Page Object Model Guide](https://playwright.dev/docs/pom)
- [Best Practices](https://playwright.dev/docs/best-practices)
- [Test Scenarios](./TEST_SCENARIOS.md)
