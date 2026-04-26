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
