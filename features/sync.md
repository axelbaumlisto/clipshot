---
layout: default
title: How sync works
nav_order: 1
parent: How it works
---
### Clipboard → Peers → File

In normal use, Clipshot watches your clipboard.

When you copy something:
- text is synced as text
- images and files are saved locally and shared to peers
- peers receive the same item and store it locally

### File storage (`~/.clipshot/sync/`)

Clipshot keeps synced files in a local folder:

```text
~/.clipshot/sync/
```

This folder is where received files and synced clipboard assets are stored.

### Catch-up on reconnect

When a device reconnects, Clipshot can send recent clipboard history it missed.

How much it catches up is controlled by:
- **Settings → Sync → Catch-up sync limit**

### Outbox for offline peers

If another device is temporarily offline, Clipshot keeps recent syncable items long enough to retry when that device comes back.

In practice, this means you do not always need to recopy the same item after a short disconnect.

### Mesh forwarding

If device A can't reach device C directly, but both can reach device B — data flows through B automatically. No configuration needed. This works for text, images, and files up to 200 MB.

Each forwarded message carries an origin ID and hop count (TTL). Content is deduplicated — the same item is never delivered twice. Maximum 5 hops.

### Encryption & Security

All data travels over **QUIC with TLS 1.3** encryption — the same technology used by Google and Cloudflare. Content is encrypted in transit between your devices.

**What's encrypted:**
- All clipboard content (text, images, files)
- Peer-to-peer handshake and data transfer
- Relay-forwarded traffic (relay sees encrypted packets, not content)

**What's NOT encrypted at rest:**
- Files in `~/.clipshot/sync/` are stored unencrypted on disk
- Settings in `~/.config/clipshot/` are plaintext TOML

**Group isolation:**
- Each account has a `group_token` — only devices with the same token can discover each other
- Peer relay only forwards traffic within the same group
- A node from another account on the same network **cannot** see your devices, intercept your data, or use your node as a relay
- mDNS discovery verifies group membership during handshake — mismatched nodes are rejected
- The portal never sees clipboard content — only device metadata for discovery

**Relay privacy:**
- Central relay (relay.clipshot.cc) forwards encrypted QUIC packets — it cannot read content
- Peer relay (embedded on each node) only serves group members
- N0 public relay (iroh) — same: encrypted passthrough, no content access
