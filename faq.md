---
layout: default
title: FAQ
nav_order: 8
---

## What exactly does Clipshot do?

Clipshot watches your clipboard. When you copy text, a screenshot, or a file — it instantly sends that content to all your other devices. They receive it and place it in their clipboard (or save it as a file). Copy on one machine, paste on another.

## Do I need an account to use Clipshot?

No. On first launch, click **Use on local network only** — devices find each other via mDNS or local pair codes (`LOCAL_WORD_42`). An account adds cloud device discovery and relay for different networks, but it's optional.

## Does my data go through your servers?

No. Clipboard content travels directly between your devices over encrypted QUIC connections. The portal only helps devices find each other — it never sees your clipboard data. Think of it as a phone book, not a mailman.

## Is my data encrypted?

Yes. All data travels over QUIC with TLS encryption. Content is encrypted in transit between your devices. The portal servers only handle device discovery, not clipboard content.

For maximum security, Clipshot supports **post-quantum key exchange** (ML-KEM + X25519 hybrid) as an opt-in build flag (`cargo build --features pq`). This protects against future quantum computer attacks on key exchange.

## What can I sync?

Everything your clipboard can hold: text (passwords, code, URLs), images (screenshots, photos), and files up to 10 MB (200 MB with Pro). If you can copy it, Clipshot can sync it.

## How do I add a new device?

Three ways:

1. **Local pair code** — one device generates `LOCAL_MOON_42`, the other enters it. Works on same network or over Tailscale/VPN. No account needed.
2. **Portal pair code** — `BLUE-FISH-42`, works over the internet. Requires account.
3. **One-liner install**:
```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=LOCAL_MOON_42 --addr=192.168.1.10:18080
```

## What if my devices are on different networks?

On the same network, devices connect directly. If you use Tailscale, WireGuard, or any VPN — Clipshot connects through it automatically, even on the free plan. For other cases, Clipshot uses free public relay servers (iroh N0, 4 global regions). Pro adds our priority central relay for maximum speed.

## What happens if a device goes offline?

Clipshot keeps recent items in an outbox. When the device reconnects, it catches up automatically — no data lost. The mesh network forwards content through available peers, so even if two devices can't reach each other directly, data flows through a third one.

## Can I use Clipshot on a headless server?

Yes. Run `clipshot daemon --port 19231` on any Linux server — no display required. One-liner install:
```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=LOCAL_MOON_42 --addr=192.168.1.10:18080
```

## What's the difference between Free and Pro?

Free gives you the full experience for **3 devices** — all features, no restrictions on speed. Files up to 10 MB. You can sync over local network, VPN, or free public relays. Pro removes the device limit, increases file size to 200 MB, adds our priority relay, 30-day clipboard history, and direct support.

## Why should I pay if it works for free?

Running Clipshot takes time and resources — we maintain relay servers, build apps for 3 platforms, and provide support. The free plan is fully functional. If Clipshot supports your work professionally, we ask you to upgrade. Your subscription keeps the project alive for everyone.

## Can I use Clipshot over Tailscale or WireGuard?

Yes. If your devices share a VPN network (Tailscale, WireGuard, ZeroTier), Clipshot connects through it automatically — even on the free plan. No relay needed, no extra config.

## How do updates work?

Desktop builds prompt for updates in-app. CLI installs can run `clipshot update` to check for a newer release and install it.

## Can other users on the same network see my data?

No. Every account has a unique group token. Only devices with the same token can discover each other. A stranger's Clipshot node on the same WiFi cannot see your devices, read your clipboard, or route traffic through your node. Peer relay only serves your own group. mDNS discovery rejects mismatched groups during handshake.

## Does the relay server see my clipboard?

No. Relay servers (both central and peer relays) forward encrypted QUIC packets. They cannot decrypt or read the content. Think of it as a sealed envelope passing through a post office — the envelope is opaque.

## Can I get a refund?

Yes. Full refunds within 60 days of payment, no questions asked. Email us.

## Where are my files and settings stored?

Synced files: `~/.clipshot/sync/`. Settings: `~/.config/clipshot/settings.toml`.
