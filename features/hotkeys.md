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
- **Manage Account**
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

Note: the Dock menu does not include **Copy Last Synced** or **Manage Account**. Like the tray menu, the Dock menu uses a static **Toggle Sync** label.

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

### Cmd+B / Ctrl+B: Paste as Path

When you copy an image or file, Clipshot syncs it and saves a local copy at `~/.clipshot/sync/img_YYYYMMDD_XXXX.png`. The clipboard still holds the original image/file data, so two different paste shortcuts now do different things:

| Shortcut | Pastes | Use case |
| --- | --- | --- |
| **Cmd+V** / **Ctrl+V** | the **image / file data** (normal clipboard paste) | dropping a screenshot inline in chat, docs, or an editor |
| **Cmd+B** / **Ctrl+B** | the **file path** as text (e.g. `~/.clipshot/sync/img_20260502_339b.png`) | attaching a file, uploading, or referencing it in a terminal command |

Default hotkey: **Cmd+B** on macOS, **Ctrl+B** on Windows/Linux.

How it works:
1. You copy an image or file → Clipshot syncs it and saves to `~/.clipshot/sync/`.
2. Press **Cmd+B** → Clipshot saves your current clipboard, temporarily writes the path into the clipboard, simulates **Cmd+V / Ctrl+V**, then restores the original image/file content.
3. The path is pasted as text into the active app — works in editors, terminals, chat apps, file dialogs.

Example workflow (screenshot → chat → terminal):

1. Take a screenshot — image is on your clipboard, Clipshot saves it to `~/.clipshot/sync/img_20260502_339b.png`.
2. **Cmd+V** in a chat app → screenshot is pasted inline as an image.
3. **Cmd+B** in your terminal → `~/.clipshot/sync/img_20260502_339b.png` is pasted as text, ready for `scp`, `open`, `mv`, etc.

Change or disable the hotkey:
- **GUI:** Settings → General → **Paste Path Hotkey** dropdown — pick a different combo or the blank option to disable.
- **`settings.toml`:** `paste_path_hotkey = "cmd+b"` (or any combo); empty string / removed line disables it.
- Restart Clipshot after changing.

The hotkey works in the desktop GUI and in `clipshot daemon` headless mode.

**macOS permissions required for hotkeys:**

| Permission | Where to grant | What breaks without it |
|---|---|---|
| **Input Monitoring** | System Settings → Privacy & Security → Input Monitoring | All global hotkeys silently stop working (Cmd+Shift+S, Cmd+B) |
| **Accessibility** | System Settings → Privacy & Security → Accessibility | Cmd+B cannot simulate the paste keystroke |

Steps to grant:
1. Open **System Settings → Privacy & Security → Input Monitoring**, click **+**, add Clipshot.
2. Open **System Settings → Privacy & Security → Accessibility**, click **+**, add Clipshot.
3. **Restart Clipshot** after granting either permission.

When a permission is missing, an amber **PermissionBanner** appears at the top of the app. Click the banner to jump directly to the relevant System Settings pane, or click **Restart** once you have granted the permission. The banner clears automatically once both permissions are granted.

### Header: Sync toggle button

The top header shows a sync state pill and an icon-only toggle button.

Status labels in the pill:
- **Ready**
- **Sending**
- **Receiving**
- **No peers**
- **Paused**

Button icons:
- when sync is active, the button shows a **pause** icon — click to pause
- when sync is paused, the button shows a **play** icon — click to resume
- during a transfer, the button shows a **spinner** icon
- when the state is **No peers**, the button is hidden

### Clipboard Toggle Hotkey

Default: **none** (not set). Toggles clipboard between image/file content and the sync file path. Useful for pasting images directly into chat apps. Configure via `clipboard_hotkey` in `settings.toml` (not exposed in the GUI). Example: `clipboard_hotkey = "ctrl+a"`.
