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

### Local Pair Code (no account needed)

A Local Pair Code (`LOCAL_MOON_42`) lets you pair two devices without any account or portal.

Flow:
1. On device A: `clipshot pair --local` → get `LOCAL_MOON_42`
2. On device B: `clipshot pair --local LOCAL_MOON_42 --addr 192.168.1.10:18080`
3. Done — devices sync automatically

Best for:
- first-time setup without account
- same network or Tailscale/VPN
- headless servers

### Portal Pair Code (requires account)

A Portal Pair Code (`BLUE-FISH-42`) is the easiest way to put a new device into the same private group.

Best for:
- devices on different networks
- using the one-line installer
- adding devices to an existing account

### Share Link (`clipshot://`)

A `clipshot://` link is a shareable connection link.

Typical flow:
1. Generate it with `clipshot share-uri`.
2. On another device, open **Peers → Add Device → Advanced → Paste Link**.
3. Paste the link and click **Connect**.

### Local Network Scan (mDNS, LAN only)

Clipshot scans the local network for nearby devices using mDNS (`_clipshot._tcp.local.`).

**Automatic mode:** When `auto_discover=true` and connected peers < free limit, the daemon scans every 60 seconds and auto-adds discovered peers.

**Manual mode:** In the GUI, open **Pair → Scan LAN** to scan and select devices.

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
| Hub/Portal | automatic discovery | Yes | Yes | nothing after setup |
| Local Pair Code | first setup, no account | No | **No** | `LOCAL_MOON_42` |
| Portal Pair Code | adding across networks | Yes | Yes | `BLUE-FISH-42` |
| Share Link | advanced sharing | Usually | No | `clipshot://...` |
| Local Network Scan | same LAN | No | No | just click Scan (auto every 60s) |
| Manual Address | expert/manual setup | No | No | address + optional password |
