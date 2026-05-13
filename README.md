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
1. On device A: `clipshot pair` → shows code like `482 917`
2. On device B: `clipshot pair 482917`
3. Compare the 4 confirmation digits on both screens → Done

Clipboards sync automatically. No account, no server, no configuration.

**Optional: register for web dashboard:**
```bash
clipshot setup   # opens browser → sign in with Google/GitHub/email → done
```

## Features

- **Automatic clipboard sync** — copy text/images/files, synced to all connected devices
- **P2P mesh** — devices connect directly via QUIC (iroh), no cloud relay needed
- **Works without internet** — local network (mDNS), share URI, or manual add
- **GUI + CLI** — desktop app with tray icon, or headless daemon for servers
- **Hub discovery** — devices find each other via Portal without an account (group token auto-created at first pair)
- **Pair codes** — one 6-digit numeric code, no account needed, DH-verified
- **Catch-up** — missed syncs delivered on reconnect
- **Paste as path** — `Cmd+B` / `Ctrl+B` pastes the synced file path instead of image data (great for chat attachments, terminal commands, file uploads)
- **Cross-platform** — macOS (DMG), Linux (binary), Windows (NSIS installer or portable binary)
- **Transfer indicator** — header shows live progress while syncing (↑ img.png 450KB/1.2MB)
- **`install.sh --token`** — install + join a group in one command: `curl ... | bash -s -- --token=clip_xxx`

## Usage

```bash
# Desktop (auto-launches GUI if display available)
clipshot

# Headless daemon
clipshot daemon --port 19231 --http-port 15282

# CLI commands
clipshot pair              # Generate 6-digit pair code
clipshot pair 482917       # Join with code from other device
clipshot share-uri         # Generate share link
clipshot push "hello"      # Send text to peers
clipshot history           # Show sync history
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
| Rust unit | 1637 | `cargo test --lib` |
| Vitest | 271 | `cd web && bun x vitest run` |
| Docker API E2E | 43 | `bun x playwright test --project=docker-api` |
| Stress | 500 | `tests/e2e/scripts/overnight-pair-v2-stress.sh` |

## License

MIT
