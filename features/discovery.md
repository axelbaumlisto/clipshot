---
layout: default
title: How devices find each other
nav_order: 2
parent: How it works
---
### Hub/Portal (automatic, works everywhere)

This is the normal way Clipshot works.

If two devices share the same:
- Hub Portal URL
- Group Token

then Clipshot Portal helps them discover each other automatically.

Best for:
- home + office + laptop combinations
- remote servers
- devices on different networks

### Pair Code (recommended for new devices)

A Pair Code is the easiest way to put a new device into the same private group.

Best for:
- first-time setup
- adding your own devices
- using the one-line installer

### Share Link (`clipshot://`)

A `clipshot://` link is a shareable connection link.

Typical flow:
1. Generate it with `clipshot share-uri`.
2. On another device, open **Peers → Add Device → Advanced → Paste Link**.
3. Paste the link and click **Connect**.

### Local Network Scan (mDNS, LAN only)

This method looks for nearby Clipshot devices on the same local network.

Best for:
- quick local setup
- devices on the same Wi‑Fi or Ethernet network

### Manual Address

If you already know a device address, add it manually.

Use this when:
- you have an `iroh://` address
- you know the device IP or hostname
- the node requires a password

### Comparison table

| Method | Best for | Internet needed | Works across different networks | What you enter |
|---|---|---:|---:|---|
| Hub/Portal | everyday automatic discovery | Yes | Yes | nothing after setup |
| Pair Code | adding a new device | Yes | Yes | short code |
| Share Link | advanced sharing | Usually | Yes | `clipshot://...` |
| Local Network Scan | same LAN | No | No | just click Scan |
| Manual Address | expert/manual setup | No | Sometimes | address + optional password |
