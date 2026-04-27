---
layout: default
title: FAQ
nav_order: 8
---
## How does Clipshot work?

Clipshot watches your local clipboard, syncs text directly, and stores synced images and files in `~/.clipshot/sync/` so peers can fetch the same content. Recent items are kept long enough for reconnect catch-up.

## Is my clipboard data sent to a central server?

No clipboard payloads are meant to move peer-to-peer between your own devices. The portal is used as a control plane for discovery, pairing, and subscription state, not as the main data store.

## Which platforms are supported?

Clipshot supports macOS, Linux, and Windows. Desktop builds provide the full GUI. Headless Linux or server setups can run the daemon and CLI instead.

## How do I pair a new device?

The recommended path is **Pair Code**. On an existing device, open **Pair device** and generate a code. On the new device, enter that code in the welcome screen or run `clipshot pair WORD-WORD-00`. You can also create a new account without a pair code by running `clipshot setup` or clicking **Create Account** on the Welcome Screen.

## Can I use Clipshot on a headless server?

Yes. Start it with `clipshot daemon --port 19231` and use `clipshot pair WORD-WORD-00` or `clipshot add-uri 'clipshot://node/...'` to join the mesh. Service management is available with `clipshot service install`. Run `clipshot doctor` to verify that the daemon, peers, and hub connection are healthy.

## What is the difference between Lite and Pro?

Lite is free and supports up to 2 active peer connections (so you can sync between up to 3 devices total). It includes local sync and clipboard history. Pro removes the peer limit, enables relay transport for syncing across different networks, and includes everything in Lite. A Lifetime Pro option is also available.

## How do updates work?

Desktop builds can prompt for updates in-app. CLI and headless installs can run `clipshot update` to check clipshot.cc for a newer release and install it, or `clipshot update --check` to only check availability. Re-running the install script is also a safe way to update installer-based setups.

## How do I uninstall Clipshot?

If you installed Clipshot as a service, run `clipshot service uninstall`. Then remove the binary and, if you want a full cleanup, delete your config directory and `~/.clipshot/` sync data.

## Where are my synced files and settings stored?

Synced files live in `~/.clipshot/sync/`. Settings usually live in `~/.config/clipshot/settings.toml` or the platform-equivalent app config directory.

### Can I use Clipshot over Tailscale or WireGuard?

Yes. If your devices share a VPN network (Tailscale, WireGuard, ZeroTier), Clipshot will connect through it automatically — even on the free plan. No relay needed, no extra config. Clipshot detects all network interfaces including VPN IPs.
