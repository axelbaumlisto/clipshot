---
layout: default
title: Installation
nav_order: 2
---
Use the one-line installer if you want the fastest setup. Use the binary or source build paths if you prefer to control the install manually.

## Install Clipshot

### One-liner (recommended)

The fastest way to add a new Linux or macOS device is:

1. On a device that is already connected, open **Pair device** or run `clipshot pair`.
2. Generate a **Pair Code**.
3. On the new device, run:

```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=YOUR-PAIR-CODE
```

Replace `YOUR-PAIR-CODE` with the code from step 2.

What the installer does:
- downloads the correct binary for your OS and CPU
- saves initial settings
- joins your private Clipshot group with the pair code
- installs Clipshot as a background service
- starts the daemon automatically

Supported by the installer script:
- Linux
- macOS

For Windows, use the binary download method below.

### Download binary

Download the binary that matches your machine:

- `clipshot-linux-x64`
- `clipshot-macos-x64`
- `clipshot-macos-arm64`
- `clipshot-windows-x64.exe`

What happens next:
- On a desktop system with a display, running `clipshot` opens the GUI.
- On a headless machine, run `clipshot daemon` to start the background sync service.

### Build from source

If you prefer to build Clipshot yourself:

```bash
git clone <repo>
cd clipshot
cd web && bun install && bun run build && cd ..
cargo build --release
```

Headless build (no GUI):

```bash
cargo build --release --no-default-features --features iroh
```

### System requirements

| Platform | Clipboard tool | Notes |
|---|---|---|
| Linux X11 | `xclip` | Required for clipboard access |
| Linux Wayland | `wl-clipboard` | Required for clipboard access |
| Linux Wayland hotkey | XWayland recommended | Global hotkey support is limited on pure Wayland |
| macOS | `pngpaste` recommended | Strongly recommended for images |
| Windows | none | Built-in clipboard support |

Notes:
- On macOS, `pngpaste` is highly recommended. Without it, copied images can become huge raw TIFF files and may fail to sync.
- On Linux, if clipboard tools are missing, file sync may still work but live clipboard sync will not.

## First launch

If Clipshot has no group token yet, it opens the **Welcome Screen** instead of the full dashboard.

### Welcome Screen

![Welcome Screen](docs/images/welcome.png)

The Welcome Screen shows:
- a short 3-step intro: **Pair → Connect → Sync**
- a primary button: **Pair with another device**
- a collapsible section: **Enter token manually**
- a link: **Open Settings**

![Welcome Screen annotated](docs/images/welcome-annotated.png)

The callouts above show: ① **Pair with another device** — the recommended way to join. ② **Enter token manually** — expand to paste an existing group token. ③ **Open Settings** — access settings before connecting.

This screen is for joining an existing Clipshot group.

### Pairing your first device

Recommended flow:

1. On an already connected device, open **Pair device** from the sidebar or **Add Device** from the Peers page.
2. Go to **Pair Code** and click **Generate Code**.
3. Clipshot shows a code like `WORD-WORD-00`.
4. On the new device, open Clipshot and click **Pair with another device**.
5. Enter the code and click **Join**.
6. Clipshot shows **Paired!** and restarts.
7. After restart, the main app opens with the full dashboard.

About the generated code:
- it is valid for **5 minutes**
- the dialog also lets you copy:
  - `clipshot pair WORD-WORD-00`
  - the installer command with `--code=...`

### Alternative: enter token manually

If you already have a group token:

1. Expand **Enter token manually**.
2. Paste your token, usually starting with `clip_`.
3. Click **Save token**.
4. Clipshot restarts and opens the normal app.
