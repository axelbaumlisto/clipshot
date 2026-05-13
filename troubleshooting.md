---
layout: default
title: Troubleshooting
nav_order: 7
---
### Hotkeys not working (macOS)

If **Cmd+Shift+S** (toggle sync) or **Cmd+B** (paste path) do nothing on macOS, the cause is almost always a missing macOS permission.

**Check the PermissionBanner first**

If any permission is missing, an amber banner appears at the top of the Clipshot window. Click the banner to open System Settings directly.

**Grant permissions manually:**

1. **System Settings → Privacy & Security → Input Monitoring**
   - Click **+**, navigate to `/Applications/Clipshot.app`, and add it.
   - Required for: all global hotkeys (Cmd+Shift+S, Cmd+B).

2. **System Settings → Privacy & Security → Accessibility**
   - Click **+**, navigate to `/Applications/Clipshot.app`, and enable the toggle.
   - Required for: the paste simulation step in Cmd+B.

3. **Restart Clipshot** after granting either permission.

**After an app update:**

macOS invalidates TCC permission entries when the app binary changes. Clipshot detects this on startup and resets the stale entries automatically — you may see the permission prompt again after an update. Re-grant the permissions and restart.

**Headless mode:**

Permissions are only required in the desktop GUI. `clipshot daemon` (headless) does not need Input Monitoring or Accessibility.

### No peers connected

Check these first:
- make sure at least one other device is online
- verify both devices use the same **Group Token**
- open **Settings → Connection** and confirm the hub shows **Connected**
- open **Peers** and use **Add Device** or **Discover**
- if you are on different networks, use Pair Code or the hub instead of LAN scan
- if both devices share a VPN (Tailscale, WireGuard, ZeroTier), Clipshot detects VPN interfaces automatically — no relay or extra config needed

If needed, run:

```bash
clipshot doctor
```

### Sync paused

Signs:
- header status says **Paused**
- tray icon turns red (static red icon)
- Overview shows a **Sync is paused** banner

How to resume:
- click **Resume** in the header
- click **Resume** in Overview
- press the sync toggle hotkey (**Cmd+Shift+S** on macOS, **Ctrl+Shift+S** on Windows/Linux)
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

### Iroh not starting (no peers connecting)

If you see `Cannot connect to iroh peer: iroh not started` in logs, check that `enable_iroh` is not set to `false` in your `settings.toml`. The default is `true` — if the field is missing, iroh starts normally. Only set it explicitly if you want to disable iroh transport.

```toml
# settings.toml — remove this line or set to true
enable_iroh = true
```
