---
layout: default
title: Troubleshooting
nav_order: 7
---
### No peers connected

Check these first:
- make sure at least one other device is online
- verify both devices use the same **Group Token**
- open **Settings → Connection** and confirm the hub shows **Connected**
- open **Peers** and use **Add Device** or **Scan LAN**
- if you are on different networks, use Pair Code or the hub instead of LAN scan

If needed, run:

```bash
clipshot doctor
```

### Sync paused

Signs:
- header status says **Paused**
- tray icon turns red
- Overview shows a **Sync is paused** banner

How to resume:
- click **Resume** in the header
- click **Resume** in Overview
- press **Ctrl+B**
- use **Toggle Sync** from the tray or Dock menu

### Clipboard not syncing

Common causes:
- no connected peers
- sync is paused
- Linux clipboard tools are missing
- the copied item is larger than your max file size

Platform checks:
- Linux X11: install `xclip`
- Linux Wayland: install `wl-clipboard`
- macOS: install `pngpaste` for reliable image syncing

### Large files not arriving

Try this:
- increase **Settings → Sync → Max file size**
- adjust **Settings → Advanced → Transfer timeouts**
- keep both devices online until the transfer finishes
- retry a failed transfer from **History**

### Peer shows 'Limited'

This means the device exists, but your current Lite plan cannot keep it active.

What to do:
- remove or disconnect an old device
- click **Activate** on the device you want to use now
- or upgrade to Pro

### Where are logs

Useful places to check:
- Linux service logs:

```bash
clipshot service logs
journalctl --user -u clipshot -n 200
```

- Linux service status:

```bash
systemctl --user status clipshot
```

- macOS launch agent logs from the installer:

```text
~/Library/Logs/clipshot.log
~/Library/Logs/clipshot.error.log
```

## Data & config paths

### Settings: `~/.config/clipshot/settings.toml`

This is the main settings file on Linux and in the installer-based setup.

On some platforms, Clipshot may use the OS-equivalent config directory for the same file.

### Sync files: `~/.clipshot/sync/`

This folder stores synced files and clipboard assets.

### Peers: `~/.config/clipshot/peers.toml`

This file stores known/saved peers.

### HMAC secret: `~/.config/clipshot/hmac_secret`

This secret is used to validate Clipshot share links.
