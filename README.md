# Clipshot

P2P mesh clipboard sync across all your devices. Copy on one — paste on another.

One binary works everywhere: desktop GUI with system tray, headless CLI daemon, or one-shot SSH commands.

## Quick Start

```bash
# Install on any machine (Linux/macOS)
curl -fsSL https://clipshot.cc/install.sh | bash

# Or download installer: https://clipshot.cc/#download
# macOS: DMG, Windows: NSIS installer, Linux: binary
```

**Pair two devices (no account needed):**
1. On device A: `clipshot pair --local` → get `LOCAL_MOON_42`
2. On device B: `clipshot pair --local LOCAL_MOON_42 --addr 192.168.1.10:18080`
3. Done — clipboards sync automatically

**Or with Portal account:**
1. On device A: open GUI → Pair → Generate Code
2. On device B: enter the code (GUI or `clipshot pair BLUE-FISH-42`)

**Or create a new account:**
```bash
clipshot setup   # opens browser for registration
```

## Features

- **Automatic clipboard sync** — copy text/images/files, synced to all connected devices
- **P2P mesh** — devices connect directly via QUIC (iroh), no cloud relay needed
- **Works without internet** — local network (mDNS), share URI, or manual add
- **GUI + CLI** — desktop app with tray icon, or headless daemon for servers
- **Hub discovery** — devices find each other automatically via Portal (optional)
- **Pair codes** — zero-config pairing: Portal codes or local `LOCAL_WORD_NN` (no account)
- **Catch-up** — missed syncs delivered on reconnect
- **Cross-platform** — macOS, Linux, Windows

## Usage

```bash
# Desktop (auto-launches GUI if display available)
clipshot

# Headless daemon
clipshot daemon --port 19231 --http-port 15282

# CLI commands
clipshot pair WORD-WORD-00      # Pair via Portal
clipshot pair --local           # Generate local pair code (no account)
clipshot pair --local LOCAL_MOON_42 --addr 192.168.1.10:18080  # Join local
clipshot share-uri              # Generate share link
clipshot push "hello"           # Send text to peers
clipshot history                # Show sync history
```

## Build

```bash
cd web && bun install && bun run build && cd ..
cargo build --release                                    # GUI
cargo build --release --no-default-features --features iroh  # Headless
```

## Documentation

- **[User Guide](docs/USER_GUIDE.md)** — installation, every screen, every button, troubleshooting
- **[CLAUDE.md](CLAUDE.md)** — developer reference, architecture, API, testing
- **[E2E Testing](docs/E2E_TESTING.md)** — Docker E2E test suites

## Tests

| Suite | Count | Command |
|-------|-------|---------|
| Rust unit | 1459 | `cargo test --lib` |
| Vitest | 232 | `cd web && bun x vitest run` |
| Docker API E2E | 43 | `npx playwright test --project=docker-api` |
| Stress | 542 | `scripts/stress-test.sh 50` |

## License

MIT
