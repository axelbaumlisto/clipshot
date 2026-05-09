---
layout: default
title: System tray & keyboard shortcuts
nav_order: 3
parent: How it works
---
### Icon States

Clipshot uses five tray states:

| State | What it means | Tray look |
|---|---|---|
| Ready | Sync is enabled and idle | static blue icon |
| Sending | This device is sending clipboard content | animated orange icon |
| Receiving | This device is receiving clipboard content | animated blue icon |
| No peers | Sync is enabled but nothing is connected | static red cross |
| Paused | You paused sync manually | static red icon |

### Tray Menu

The tray icon tooltip is **Clipshot - P2P Clipboard Sync**.

Tray menu items:
- **Open Dashboard**
- **Copy Last Synced**
- **Toggle Sync**
- **Upgrade to Pro** on Lite plans
- **Quit**

Platform behavior:
- macOS: the tray menu opens on right-click
- Linux and Windows: the tray menu opens on left-click

### macOS Dock Menu

On macOS, the Dock icon has its own menu:
- **Open Dashboard**
- **Toggle Sync**
- separator
- **Quit**

Note: the Dock menu does not include **Copy Last Synced** or **Upgrade to Pro**.

### Dock Progress Bar

During active transfers, Clipshot shows overall transfer progress on the app window taskbar/dock indicator where the platform supports it.

What it does:
- shows combined progress for all active transfers
- clears automatically when transfers finish
- on macOS, Clipshot also prevents idle sleep while a transfer is running

## Keyboard shortcuts

### Cmd+Shift+S / Ctrl+Shift+S: Toggle Sync

By default, **Cmd+Shift+S** (macOS) or **Ctrl+Shift+S** (Windows/Linux) pauses or resumes clipboard sync.

You can change this shortcut in **Settings → General → Toggle Sync Hotkey**.

If the hotkey changes, restart Clipshot.

### Cmd+B / Ctrl+B: Paste File Path

By default, **Cmd+B** (macOS) or **Ctrl+B** (Windows/Linux) pastes the file path of the last synced clipboard content.

How it works:
1. You copy an image or file → Clipshot syncs it and saves to `~/.clipshot/sync/`
2. Press **Cmd+B** → Clipshot temporarily puts the file path in clipboard, simulates Ctrl+V/Cmd+V, then restores original clipboard
3. The path (e.g. `~/.clipshot/sync/img_20260502_339b.png`) is pasted into your editor/terminal

### Header: Pause/Resume button

The top header shows a sync status pill plus an action button.

Status labels you may see:
- **Ready**
- **Sending**
- **Receiving**
- **No peers**
- **Paused**

Button behavior:
- when sync is active, the button says **Pause**
- when sync is paused, the button says **Resume**
- when the state is **No peers**, only the status pill is shown

### Clipboard Toggle Hotkey

Default: **none** (not set). Toggles clipboard between image/file content and the sync file path. Useful for pasting images directly into chat apps. Configure via `clipboard_hotkey` in `settings.toml` (not exposed in the GUI). Example: `clipboard_hotkey = "ctrl+a"`.
