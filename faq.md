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

The recommended path is **Pair Code**. On an existing device, open **Pair device** and generate a code. On the new device, enter that code in the welcome screen or run the one-line installer with `--code=WORD-WORD-00`.

## Can I use Clipshot on a headless server?

Yes. Start it with `clipshot daemon --port 19231` and use `clipshot pair WORD-WORD-00` or `clipshot add-uri 'clipshot://node/...'` to join the mesh. Service management is available with `clipshot service install`.

## What is the difference between Lite and Pro?

Lite is free and supports up to 2 devices with local sync and history. Pro removes the device cap, enables relay transport, and includes everything in Lite. A Lifetime Pro option is also available.

## How do updates work?

Desktop builds can prompt for updates in-app. CLI and headless installs can check or install updates with `clipshot update` when the updater is available. Re-running the install flow is also a safe fallback for installer-based setups.

## How do I uninstall Clipshot?

If you installed Clipshot as a service, run `clipshot service uninstall`. Then remove the binary and, if you want a full cleanup, delete your config directory and `~/.clipshot/` sync data.

## Where are my synced files and settings stored?

Synced files live in `~/.clipshot/sync/`. Settings usually live in `~/.config/clipshot/settings.toml` or the platform-equivalent app config directory.
