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

### Pair Code (6-digit, works everywhere)

A 6-digit numeric pair code is the easiest way to connect two devices. No account required — the first pair automatically creates a local group identity.

Flow:
1. On device A: `clipshot pair` → shows code like `482 917`
2. On device B: `clipshot pair 482917`
3. Both devices display 4 confirmation digits — compare them to verify no MITM
4. Done — devices sync automatically

The pair flow uses the fastest available transport automatically:
- Portal relay (works across the internet, no account required for pairing itself)
- mDNS (same LAN, works without internet)

Best for:
- any first-time setup
- adding devices across different networks
- using the one-line installer (`--code=482917`)
- headless servers (same LAN, Tailscale, or any routable network)

### Share Link (`clipshot://`)

A `clipshot://` link is a shareable connection link.

Typical flow:
1. Generate it with `clipshot share-uri`.
2. On another device, run `clipshot add-uri 'clipshot://...'` or open **Peers → Add Device** and enter the URI.
3. The devices are connected.

### Local Network Scan (mDNS, LAN only)

Clipshot scans the local network for nearby devices using mDNS (`_clipshot._tcp.local.`).

**Automatic mode:** When `auto_discover=true` and connected peers < free limit, the daemon scans every 60 seconds and auto-adds discovered peers.

**Manual mode:** In the GUI, click the **Discover** button on the Peers page to trigger an immediate LAN scan.

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

| Method | Best for | Internet needed | Account needed | What you enter |
|---|---|---:|---:|---|
| Hub/Portal | automatic discovery | Yes | Optional | nothing after setup |
| Pair Code | any new device | Best with; mDNS fallback | **No** | 6-digit code (`482917`) |
| Share Link | advanced sharing | Usually | No | `clipshot://...` |
| Local Network Scan | same LAN | No | No | just click Scan (auto every 60s) |
| Manual Address | expert/manual setup | No | No | address + optional password |
