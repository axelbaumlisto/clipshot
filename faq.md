---
layout: default
title: FAQ
nav_order: 4
---

# Frequently Asked Questions

## How does Clipshot work?

Clipshot creates peer-to-peer encrypted connections between your devices using the iroh protocol (QUIC/UDP). When you copy something on one device, it's instantly sent to all connected peers.

## Is my data sent to any server?

**No.** Your clipboard data travels directly between your devices. The hub portal is only used for device discovery and pairing — it never sees your clipboard content.

## What's the difference between Lite and Pro?

| | Lite (Free) | Pro ($5/mo) |
|---|:---:|:---:|
| Devices | 2 | Unlimited |
| Local sync | ✅ | ✅ |
| Relay | — | ✅ |
| History | ✅ | ✅ |

## Can I use Clipshot on a headless server?

Yes. Run:
```bash
clipshot daemon --port 19231 --http-port 15282
```

Access the web UI at `http://localhost:15282` or use the HTTP API.

## How do I sync across different networks?

With **Pro**, the relay transport routes encrypted data through our relay server so devices don't need to be on the same network.

## What file types are supported?

Text, images (PNG, JPEG), and files of any type up to your configured max size (default: 200MB).

## How do I update?

- **GUI**: Automatic background updates
- **CLI**: Run `clipshot update`

## How do I uninstall?

- **macOS**: Delete `/Applications/clipshot.app` and `~/.clipshot/`
- **Linux**: Remove the binary and `~/.clipshot/`
- **systemd**: `clipshot service uninstall` then remove binary
