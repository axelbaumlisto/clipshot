---
layout: default
title: Home
nav_order: 1
---
Clipshot keeps text, images, and files moving between your own devices without a central storage account.

[Download](https://clipshot.cc/#download){: .btn .btn-primary }
[Installation](installation/){: .btn }
[Application guide](gui/){: .btn }

## Quick start

```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=WORD-WORD-00
```

1. On a device that is already connected, open **Pair device** or run `clipshot pair`.
2. Generate a **Pair Code**.
3. Run the installer on the new device.
4. After pairing, Clipshot opens the full dashboard and keeps syncing automatically.

## What is Clipshot

Clipshot is a peer-to-peer clipboard sync app for your own devices. It keeps text, images, and files moving between your laptop, desktop, and servers without a central storage account.

## What you can do

- Sync text, images, and files across your own devices
- Pair new machines with a short **Pair Code**
- Run a desktop GUI, a browser UI, or a headless daemon
- Catch up after reconnects instead of recopied clipboard items
- Use direct peer-to-peer transport with relay fallback when needed

## Read next

- [Installation & first launch](installation/)
- [Overview page](gui/overview/)
- [Peers page](gui/peers/)
- [How sync works](features/sync/)
- [Troubleshooting](troubleshooting/)

<img src="docs/images/overview.png" alt="Overview page" class="img-wide">
