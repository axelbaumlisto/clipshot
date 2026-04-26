<p align="center">
  <img src="https://clipshot.cc/favicon.svg" width="80" alt="Clipshot Logo">
</p>

<h1 align="center">Clipshot</h1>

<p align="center">
  <b>Cross-platform P2P clipboard sync</b><br>
  Copy on one device, paste on another. Encrypted, no cloud, no accounts.
</p>

<p align="center">
  <a href="https://clipshot.cc">Website</a> •
  <a href="https://clipshot.cc/#pricing">Pricing</a> •
  <a href="docs/USER_GUIDE.md">User Guide</a> •
  <a href="https://clipshot.cc/#download">Download</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/version-0.5.0-emerald" alt="Version">
  <img src="https://img.shields.io/badge/platforms-macOS%20%7C%20Linux%20%7C%20Windows-blue" alt="Platforms">
  <img src="https://img.shields.io/badge/tests-2400%2B%20passing-brightgreen" alt="Tests">
  <img src="https://img.shields.io/badge/license-proprietary-lightgrey" alt="License">
</p>

---

## What is Clipshot?

Clipshot syncs your clipboard across all your devices in real-time using peer-to-peer encrypted connections. No cloud servers see your data.

**Key features:**
- 🔒 **P2P Encrypted** — data never touches our servers
- ⚡ **Instant Sync** — clipboard changes propagate in milliseconds
- 🖥️ **Cross-Platform** — macOS, Linux, Windows, headless servers, WSL
- 🌐 **Relay Transport** (Pro) — sync across any network
- 🔄 **Auto-Update** — seamless background updates
- 📋 **Clipboard History** — browse and restore past items

## Install

### macOS
```bash
brew install --cask clipshot
```
Or download from [clipshot.cc](https://clipshot.cc/#download)

### Linux
```bash
curl -fsSL https://clipshot.cc/install.sh | bash
```

### One-liner with pairing
```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=WORD-WORD-00
```

### Headless / Server
```bash
clipshot daemon --port 19231 --http-port 15282
```

## Quick Start

1. **Install** on two or more devices
2. **Pair** using a pair code or share link
3. **Copy** on one device — it appears on all others instantly

## Screenshots

<p align="center">
  <img src="docs/images/overview.png" width="600" alt="Overview">
</p>

## Documentation

- [User Guide](docs/USER_GUIDE.md) — complete usage guide with screenshots
- [FAQ](docs/FAQ.md) — frequently asked questions
- [Changelog](CHANGELOG.md) — version history

## Pricing

| | Lite (Free) | Pro ($5/mo) |
|---|---|---|
| Devices | 2 | Unlimited |
| Local sync | ✅ | ✅ |
| Relay transport | — | ✅ |
| Clipboard history | ✅ | ✅ |
| Auto-update | ✅ | ✅ |
| Priority support | — | ✅ |

**Lifetime Pro: $200** — one-time purchase, everything in Pro forever.

[Get started →](https://clipshot.cc/#pricing)

## Support

- 📧 [contact@clipshot.cc](mailto:contact@clipshot.cc)
- 🐛 [GitHub Issues](https://github.com/clipshot1/clipshot/issues)
- 🌐 [clipshot.cc](https://clipshot.cc)

## License

Clipshot is proprietary software. Free to download and use under the [EULA](LICENSE).
