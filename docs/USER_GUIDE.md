# Clipshot — User Guide

## What is Clipshot

Clipshot is a peer-to-peer clipboard sync app for your own devices. It keeps text, images, and files moving between your laptop, desktop, and servers without a central storage account.

## Installation

### One-liner (recommended)

**With Portal (pair code):**

```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=WORD-WORD-00
```

Get the code from another device: GUI → Pair → Generate Code, or `clipshot pair`.

**Without Portal (share link):**

```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --uri='clipshot://node/...'
```

Get the link from another device: `clipshot share-uri` or GUI → Peers → Share Link.

**Install only (connect later):**

```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --headless --no-autostart
clipshot pair WORD-WORD-00       # with Portal
clipshot add-uri 'clipshot://...' # without Portal
clipshot setup                    # create new account in browser
```

What the installer does:
- downloads the correct binary for your OS and CPU
- creates the config directory
- makes the binary executable

Supported by the installer script:
- Linux
- macOS

For Windows, download the NSIS installer from [clipshot.cc](https://clipshot.cc).

install.sh flags:
- `--code=CODE` — pair code: `BLUE-FISH-42` (Portal) or `LOCAL_MOON_42` (local, no Portal)
- `--uri=URI` — share link from another device (no Portal needed)
- `--addr=IP:PORT` — target device address (for local pair codes)
- `--headless` — download bare binary only (no DMG/GUI)
- `--no-autostart` — skip systemd/launchd service creation
- `--port=PORT` — daemon port (default 19231)

Routing: codes starting with `LOCAL_` are resolved locally (HTTP API), others via Portal.

### Download installer

Download the installer for your platform from [clipshot.cc](https://clipshot.cc):

| Platform | File | Type |
|---|---|---|
| macOS ARM (M1+) | `Clipshot_VERSION_aarch64.dmg` | DMG installer |
| macOS Intel | `clipshot-macos-x64` | Binary |
| Windows x64 | `Clipshot_VERSION_x64-setup.exe` | NSIS installer |
| Linux x64 | `clipshot-linux-x64` | GUI Binary (requires GTK3 + WebKit2) |

**macOS DMG:**
1. Open the DMG
2. Drag Clipshot to Applications
3. **First launch:** right-click the app in Applications → **Open** (do not double-click)
4. macOS will warn that the app is from an unidentified developer — click **Open** to confirm
5. After the first launch, double-click works normally

> **Why the warning?** Clipshot is not yet notarized with Apple. The app is ad-hoc signed, which means macOS Gatekeeper blocks it on first launch. Right-click → Open bypasses this check once. This will be resolved when Clipshot is notarized through the Apple Developer Program.

**If macOS says the app is "damaged" or "cannot be opened":**

This happens when macOS quarantine flags remain on the app bundle. Run this in Terminal:

```bash
xattr -cr /Applications/Clipshot.app
```

Then launch normally. The `-cr` flag recursively clears all extended attributes including the quarantine flag.

**macOS binary (non-DMG, e.g. from `install.sh --headless`):**

If you downloaded the bare binary instead of the DMG, clear quarantine before the first run:

```bash
xattr -cr ./clipshot
chmod +x ./clipshot
./clipshot
```

**Summary of macOS Gatekeeper workarounds:**

| Symptom | Fix |
|---|---|
| "Clipshot can't be opened because it is from an unidentified developer" | Right-click → Open → Open |
| "Clipshot is damaged and can't be opened" | `xattr -cr /Applications/Clipshot.app` |
| Binary downloaded via curl/browser won't run | `xattr -cr ./clipshot && chmod +x ./clipshot` |
| Repeated warnings after update | Re-run `xattr -cr` after each update until notarization is available |

**Windows NSIS:**
1. Run the installer (double-click or `Clipshot_VERSION_x64-setup.exe /S` for silent)
2. Launch from Start Menu
3. Allow through Windows Firewall when prompted

**Linux / macOS Intel binary:**
```bash
chmod +x clipshot-linux-x64
./clipshot-linux-x64
```

On a desktop system with a display, running `clipshot` opens the GUI.
On a headless machine, run `clipshot daemon` to start the background sync service.

### Build from source

If you prefer to build Clipshot yourself:

```bash
git clone <repo>
cd clipshot
cd web
bun install
bun run build
cd ..
cargo build --release
```

Headless build (servers only, no GUI):

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

## First Launch

If Clipshot has no group token yet, it opens the **Welcome Screen** instead of the full dashboard.

### Welcome Screen

![Welcome Screen](images/welcome.png)

The Welcome Screen shows:
- a short 3-step intro: **Pair → Connect → Sync**
- a primary button: **Pair with another device**
- a secondary button: **Create Account** — opens your browser for registration (Google OAuth supported)
- a collapsible section: **Enter token manually**
- a link: **Open Settings**

![Welcome Screen annotated](images/welcome-annotated.png)

The callouts above show: ① The intro card with the 3-step overview. ② **Pair with another device** — the recommended way to join an existing group. ③ **Create Account** — opens a browser-based registration flow. ④ **Enter token manually** — expand to paste an existing group token. ⑤ **Open Settings** — access settings before connecting.

This screen is for joining an existing Clipshot group or creating a new account.

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

### Alternative: enter token manually

If you already have a group token:

1. Expand **Enter token manually**.
2. Paste your token, usually starting with `clip_`.
3. Click **Save token**.
4. Clipshot restarts and opens the normal app.

## Overview Page

![Overview page](images/overview.png)

After pairing, the main window has:
- a left sidebar with **Overview**, **Peers**, **History**, **Settings**
- a **Pair device** button in the sidebar
- a **Lite** or **Pro** badge near the app name
- a top bar with sync status, peer count, and theme toggle

The sidebar can be collapsed and expanded.

### Sidebar

![Sidebar](images/sidebar.png)

![Sidebar annotated](images/sidebar-annotated.png)

The sidebar callouts show: ① App name and plan badge (**Pro** or **Lite**). ② Collapse/expand sidebar button. ③ Navigation links: **Overview**, **Peers**, **History**, **Settings**. The active page is highlighted with a background fill.

Below the app name, the **Pair device** button opens the Add Device dialog from any page. At the bottom, the sidebar footer shows the current sync state label and peer count.

### Header Bar

![Header bar](images/header-bar.png)

![Header bar annotated](images/header-bar-annotated.png)

The header bar shows: ① Sync toggle button — an icon-only button that pauses or resumes sync. Its background color reflects the current sync state (green for ready, blue for sending, amber for no peers, grey for paused). ② Peer count badge — colored dots and a connected/total count such as `2/3`. ③ Theme toggle — switches between light and dark mode.

Note: the text sync status label (**Ready**, **Sending**, **Receiving**, **No peers**, or **Paused**) appears in the sidebar footer, not in the header bar.

![Overview page annotated](images/overview-annotated.png)

The annotated view highlights: ② Sidebar — navigation and status area. ③ Status hero card — your network health at a glance. Below the hero are three cards: Last Synced, Devices, and Recent Activity. At the bottom is a collapsible Node details section.

> Note: some callouts may be hard to see on the dark theme. The five main sections are the hero card, last synced, devices, recent activity, and node details.

### Status hero

The large top card summarizes the current state.

What you see:
- headline:
  - **All systems ready** when at least one device is connected
  - **No peers connected** when nothing is online
- summary line with:
  - connected device count
  - last sync time, if available
- action buttons:
  - **Pair**
  - **View Peers** when at least one device is connected

Extra alerts inside the card:
- **Sync is paused** banner with a **Resume** button
- a device-limit warning if your current plan does not allow more devices

### Last synced card

This card shows your most recent successful transfer.

It includes:
- an icon for image, text, or file
- the file or content name
- whether it was **Sent to** or **From** a device
- size, when known
- relative time such as `just now` or `5m ago`

If nothing has synced yet, it says: **Copy something to start syncing**.

Clicking the card opens **History**.

### Devices card

This card shows currently connected devices as compact badges.

Each badge can show:
- green online dot
- device name
- latency in milliseconds, when available

If there are more than six connected devices, the card shows `+N more`.

If none are connected, it says **No devices connected**.

Clicking the card opens **Peers**.

### Activity card

This card shows the latest activity entries.

What it includes:
- up to 3 recent items
- failures first, so problems are easier to notice
- transfer lines such as:
  - `Sent photo.png to Work Mac`
  - `Received notes.txt from Server`
- peer events such as:
  - `Laptop connected`
  - `Desktop disconnected`
- file size when known
- time of day
- **View all** link to open **History**

### Node details

At the bottom of the page there is a collapsible **Node details** section.

It shows:
- node name
- uptime
- listening port
- node ID
- copy button for the node ID

## Peers Page

![Peers page](images/peers.png)

The Peers page is where you add, view, pin, reconnect, disconnect, and remove devices.

![Peers page annotated](images/peers-annotated.png)

The annotated view highlights: ① **Search devices...** field — search by name or node ID. ② Filter buttons with counts: **All**, **Connected**, **Offline**, **Pinned**. ④ **Add Device** button — opens the pairing dialog. Device cards (not annotated in this screenshot) show name, status, transport, latency, and actions.

If you have no devices yet, the page shows an empty state card with:
- **Add Device**
- **Scan LAN**
- current **Hub** status: Connected or Disconnected
- tip text recommending a pair code

### Search & Filters

At the top of the page you get:
- a **Search devices...** field
- filter buttons with counts:
  - **All**
  - **Connected**
  - **Offline**
  - **Pinned**

Search matches:
- device name
- node ID prefix

Peer cards are grouped into sections:
- **Pinned**
- **Connected**
- **Disconnected**

### Device Card

Each card represents one device.

Visible fields and indicators:
- pin star in the top-left corner
- device name
- **Connected** or **Offline** badge
- for offline devices, optional **Last seen ...** text
- optional offline reason such as:
  - **Remote device limit reached**
  - **Free tier limit reached**
- device type:
  - Laptop
  - Desktop
  - Server
  - WSL
- transport label:
  - **Direct**
  - **Relay**
- latency, or **Latency: pending** while measuring
- activity line such as:
  - **Connected, no sync yet**
  - **Last sync: 3m ago**
  - **Added to network**
- for limited devices, a note:
  - **Free tier limit reached. Activate to swap this device in.**

If the device is transferring something right now, the card also shows:
- a small progress bar
- text like **Sending 2.1MB / 10.0MB** or **Receiving ...**

If Clipshot knows multiple addresses for the same device, the card can expand to show an address list.

Buttons and actions:
- **Details** for connected devices
- **Reconnect** for normal offline devices
- **Activate** for devices blocked by the Lite plan limit
- overflow menu with:
  - **Details** for offline devices
  - **Disconnect** for connected devices
  - **Remove** for all devices

Important:
- **Disconnect** and **Remove** use a two-step confirmation in the menu: first click changes the item to **Confirm?** for a few seconds.
- Pinning a device moves it into the **Pinned** section and makes it easy to find later.

### Adding a New Device

Use either:
- **Add Device**
- **Discover**
- the sidebar **Pair device** button

All of them open the same add-device dialog.

#### Pair Code

![Add Device — Pair Code tab](images/add-device-pair.png)

This is the recommended method.

![Add Device — Pair Code annotated](images/add-device-pair-annotated.png)

The callouts show: ① **Generate Code** — creates a short code valid for 5 minutes. ② Code input field — enter a code from another device, then click **Join**. ③ **Advanced** — expand for Paste Link and Manual Entry options.

Inside the dialog:
- existing devices can click **Generate Code**
- any device can enter a code and click **Join**
- generated codes show:
  - the code itself
  - **Valid for 5 minutes**
  - a copyable CLI command

Use Pair Code when:
- adding your own laptop or phone-sized desktop companion
- installing Clipshot on a remote machine
- you want the simplest flow

#### Local Network Scan

![Add Device — Local Network tab](images/add-device-discover.png)

The **Local Network** tab scans your LAN with mDNS.

What you can do:
- click **Scan**
- see a list of found devices
- select one or more checkboxes
- click **Add Selected**

Best for:
- devices on the same home or office network
- quick setup without sharing links

Limits:
- LAN only
- does not help across the internet

#### Paste Link

In **Advanced → Paste Link**, you can paste:
- `clipshot://...`
- `iroh://...`

Then click **Connect**.

Use this when:
- someone sent you a Clipshot share link
- you want an advanced/manual pairing method

#### Manual Entry

In **Advanced → Manual Entry**, you can enter:
- **Name**
- **Address**
- optional **Password**

The address can be:
- an `iroh://...` address
- an IP address or hostname

Use this when:
- you already know the remote address
- you are connecting to a protected node that requires a password

### Device Details

Opening **Details** shows a dialog with peer information.

View mode shows:
- device name
- **Connected** or **Offline** badge
- device type, when known
- transport:
  - **Direct • QUIC**
  - **TCP**
  - or `—`
- latency
- node ID with copy button
- last sync time
- collapsible **Addresses** section

Click **Edit** to change:
- device name
- saved addresses
- auth code

Then click **Save** or **Cancel**.

### Removing a Device

To remove a device:
1. Open the device card menu.
2. Click **Remove**.
3. Click **Confirm?** before the confirmation times out.

To temporarily stop a connected device without deleting it:
1. Open the menu.
2. Click **Disconnect**.
3. Click **Confirm?**.

## History Page

![History page](images/history.png)

History is a unified timeline of transfers and peer activity.

![History page annotated](images/history-annotated.png)

The annotated view highlights: ① **Search files, peers...** field — search by filename, peer name, or content type. ② Filter chips: **All**, **Transfers**, **Events**, **Failed** with counts. Transfer rows (visible below the filters) show file, peer, direction, type badge, size, and time. **TODAY** date headers group entries by day.

### Search

The search box lets you search by:
- filename
- peer name
- content type

### Filters: All / Transfers / Events / Failed

Use the filter row to switch between:
- **All** — everything
- **Transfers** — sent and received items
- **Events** — connect and disconnect events
- **Failed** — only failed transfers

Each filter shows its current count.

### Timeline

The timeline is grouped by date with sticky date headers, so the current day label stays visible as you scroll.

If there is no history yet, the page explains that transfers and events will appear automatically.

### Transfer Row

Every transfer row can show:
- image thumbnail preview, when available
- file or content icon for non-images
- file name or content name
- text preview for text clipboard entries
- peer name
- sent/received arrow
- content badge such as **Image**, **Text**, or **File**
- size
- time of day

Actions:
- **Copy path**
- **Open folder**
- **Retry transfer** for failed items

Failed transfers:
- show a red left border
- show a short error summary
- can expand to reveal the full error, peer details, and address

In-progress transfers:
- show a blue left border
- show a progress bar
- show transferred size, total size, and percent
- can also show chunk progress, speed, and ETA

### Event Row

Event rows show peer connection changes.

Each row includes:
- connected or disconnected icon
- text like `Laptop connected` or `Server disconnected`
- optional secondary peer details
- time of day

## Settings Page

![Settings page](images/settings.png)

The Settings page is split into collapsible cards. Clipshot remembers which sections you keep open or closed.

![Settings page annotated](images/settings-annotated.png)

The annotated view highlights: ① App name in the sidebar. The main settings area shows collapsible cards: **Connection** (Hub Portal URL, Group Token, subscription info), **General** (theme, startup, notifications, hotkey), **Sync**, **Storage**, **Network**, and **Advanced**. At the bottom are **Save Changes** / **Reset to Defaults** buttons.

If you make changes, a sticky bar appears at the bottom with:
- **Discard**
- **Save**

At the bottom of the page you also get:
- **Save Changes**
- **Reset to Defaults**

### Connection (hub_url, group_token)

The **Connection** card controls how your device joins Clipshot Portal.

Visible items:
- status badge:
  - **Not configured**
  - **Connected**
  - **Disconnected**
- **Hub Portal URL**
- **Group Token**
- subscription summary showing:
  - plan: Lite or Pro
  - peer limit
  - relay enabled or disabled

You may also see:
- `Connected · N peers in network`
- `Connecting to portal...`
- device-limit warning with **Open Portal Devices** button

About `relay_url`:
- Relay URL affects connectivity too, but in the current UI it is edited in the **Network** card, not inside **Connection**.

### Working without Portal

Clipshot Portal (clipshot.cc) provides automatic device discovery and pair codes. However, **devices can connect without it**:

| Method | How | Needs Portal? |
|---|---|---|
| **Local pair code** | `clipshot pair --local` → `LOCAL_MOON_42` → other device joins | No |
| **Share URI** | `clipshot share-uri` → send link → `clipshot add-uri 'clipshot://...'` | No |
| **Local Network (mDNS)** | Automatic on same LAN (`_clipshot._tcp.local.`) — scans every 60s | No |
| **Manual add** | Enter peer name + IP address in GUI or CLI | No |
| **Portal pair code** | `clipshot pair BLUE-FISH-42` | **Yes** |

**Local pair example (no account needed):**

```bash
# Device A:
clipshot pair --local
# → LOCAL_MOON_42 • 192.168.1.10:18080

# Device B (same LAN — auto-finds via mDNS):
clipshot pair --local LOCAL_MOON_42

# Device B (different network — specify IP):
clipshot pair --local LOCAL_MOON_42 --addr 100.64.0.5:18080

# One-liner install + pair:
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=LOCAL_MOON_42 --addr=192.168.1.10:18080
```

If Portal is down or unreachable:
- Existing connections continue working (P2P, no server in data path)
- New devices can join via Share URI, mDNS, or manual add
- Pair codes will not work until Portal is restored
- Mesh forwarding delivers data through connected peers

### General (theme, hotkey)

The **General** card includes:
- **Theme**:
  - System
  - Light
  - Dark
- **Start on login** switch
- **Show notifications** switch
- **Toggle Sync Hotkey** selector
- **Clipboard Toggle Hotkey** text field

Notes:
- the default sync toggle hotkey is **Cmd+Shift+S** (macOS) or **Ctrl+Shift+S** (Windows/Linux)
- the default paste path hotkey is **Cmd+B** (macOS) or **Ctrl+B** (Windows/Linux)
- changing hotkeys requires a restart

### Sync (catchup_limit, max_file_size)

The **Sync** card includes:
- **Max file size (MB)**
- **Catch-up sync limit**

What they mean:
- **Max file size** limits what Clipshot will send
- **Catch-up sync limit** controls how many recent clipboard entries a reconnecting device can fetch

If you set a very large file limit, Clipshot warns that large files may time out.

### Storage (sync file retention)

The **Storage** card includes:
- **Max files** — maximum number of files to keep in the sync directory (0 = unlimited)
- **Max age (days)** — delete synced files older than this many days (0 = unlimited)
- **Max size (MB)** — maximum total size of the sync directory (0 = unlimited)

These settings control automatic cleanup of `~/.clipshot/sync/`.

### Network (listen_port, max_peers)

The **Network** card includes:
- **Listen Port**
- **Max Peers**
- **Auto Discover** switch
- **Relay URL**
- **Node Password**

What they do:
- **Listen Port** is the port this device listens on for P2P traffic
- **Max Peers** caps simultaneous peer connections
- **Auto Discover** enables local network discovery
- **Relay URL** is used for NAT traversal and long-distance fallback
- **Node Password** protects your node so only devices with that password can connect

Invalid listen ports are highlighted immediately.

### Advanced (timeouts, retries)

The **Advanced** card includes:
- **Poll Interval (ms)**
- **Retry Attempts**
- **Timeout (ms)**
- **Use Browser UI** switch

It also has a **Transfer timeouts** subsection with:
- **Base Transfer Timeout (ms)**
- **Timeout per MB (ms)**
- **Chunk Timeout (seconds)**
- **Direct Send Threshold (MB)**

These settings are mainly for slow links, large files, and difficult network conditions.

### Export / Import

At the bottom of the Advanced card you can:
- **Export Settings** to a JSON file
- **Import Settings** from a JSON file

Import does not apply changes immediately. After import, click **Save**.

### Diagnostics

The diagnostics area shows live runtime information with copy buttons.

Current fields:
- node name
- node ID
- uptime
- iroh address
- hub status

Use this section when:
- support asks for your node ID
- you want to verify the app is connected to the hub
- you need your device address for manual pairing

## System Tray

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

## Keyboard Shortcuts

### Ctrl+Shift+S / Cmd+Shift+S: Toggle Sync

By default, **Cmd+Shift+S** (macOS) or **Ctrl+Shift+S** (Windows/Linux) pauses or resumes clipboard sync.

You can change this shortcut in **Settings → General → Toggle Sync Hotkey**.

If the hotkey changes, restart Clipshot.

### Cmd+B / Ctrl+B: Paste File Path

By default, **Cmd+B** (macOS) or **Ctrl+B** (Windows/Linux) pastes the file path of the last synced clipboard content.

How it works:
1. You copy an image or file → Clipshot syncs it and saves to `~/.clipshot/sync/`
2. Press **Cmd+B** → Clipshot temporarily puts the file path in clipboard, simulates Ctrl+V/Cmd+V, then restores original clipboard
3. The path (e.g. `~/.clipshot/sync/img_20260502_339b.png`) is pasted into your editor/terminal

Useful for:
- Pasting a screenshot path into a chat or document
- Referencing synced files by path in terminal commands
- Quick access to the actual file on disk

This works in any app — editors, terminals, chat apps.

### Sync status indicators

The sidebar footer shows a text label for the current sync state:
- **Ready** — sync is enabled and idle
- **Sending** — clipboard content is being sent
- **Receiving** — clipboard content is being received
- **No peers** — sync is enabled but no devices are connected
- **Paused** — sync is paused

The header bar has an icon-only toggle button:
- when sync is active, it shows a **pause** icon — click to pause
- when sync is paused, it shows a **play** icon — click to resume
- during a transfer, it shows a **spinner** icon

## CLI Mode (Headless Servers)

On machines without a desktop session, Clipshot runs in CLI mode.

### Starting the daemon

Start the background sync daemon:

```bash
clipshot daemon --port 19231
```

Useful optional flags include:
- `--hub-url`
- `--group-token`
- `--relay-url`
- `--password`
- `--http-port`

### Pairing via CLI

On an existing device, generate a pair code:

```bash
clipshot pair
```

On the new device, join with that code:

```bash
clipshot pair WORD-WORD-00
```

After joining, start the daemon or relaunch the GUI.

### Useful commands

Generate a share link for this node:

```bash
clipshot share-uri
```

Add a peer from a Clipshot link:

```bash
clipshot add-uri 'clipshot://node/...'
```

Copy text back to your local terminal clipboard over SSH:

```bash
clipshot push "hello"
```

Show recent sync history:

```bash
clipshot history
```

Set up a new account (opens browser for registration):

```bash
clipshot setup
```

Check for or install updates:

```bash
clipshot update
clipshot update --check
```

Other useful CLI commands:

```bash
clipshot status
clipshot pause
clipshot resume
clipshot toggle
clipshot list-peers
clipshot retry-transfer /path/to/file
clipshot doctor
```

### Auto-start

Clipshot starts automatically on boot by default. The one-line installer configures this for you.

| Platform | Mechanism | Config file |
|---|---|---|
| Linux | systemd user service | `~/.config/systemd/user/clipshot.service` |
| macOS (CLI) | launchd agent | `~/Library/LaunchAgents/cc.clipshot.daemon.plist` |
| macOS (GUI) | Login Items | via System Events |

Install or manage the service manually:

```bash
clipshot service install --port 19231 --http-port 18080
clipshot service status
clipshot service logs
clipshot service uninstall
```

To skip auto-start during install: `curl ... | bash -s -- --no-autostart`

To toggle from the GUI: **Settings → General → Start at login**

**PID lock**: only one daemon runs at a time. A second instance exits with "Another clipshot daemon is already running". The lock file is at `~/.config/clipshot/daemon.lock`.

## How Sync Works

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

## How Devices Find Each Other

### Hub/Portal (automatic, works everywhere)

This is the normal way Clipshot works.

If two devices share the same:
- Hub Portal URL
- Group Token

then Clipshot Portal helps them discover each other automatically.

Best for:
- home + office + laptop combinations
- remote servers
- devices on different networks

### Pair Code (recommended for new devices)

A Pair Code is the easiest way to put a new device into the same private group.

Best for:
- first-time setup
- adding your own devices
- using the one-line installer

### Share Link (`clipshot://`)

A `clipshot://` link is a shareable connection link.

Typical flow:
1. Generate it with `clipshot share-uri`.
2. On another device, open **Peers → Add Device → Advanced → Paste Link**.
3. Paste the link and click **Connect**.

### Local Network Scan (mDNS, LAN only)

This method looks for nearby Clipshot devices on the same local network.

Best for:
- quick local setup
- devices on the same Wi‑Fi or Ethernet network

### Manual Address

If you already know a device address, add it manually.

Use this when:
- you have an `iroh://` address
- you know the device IP or hostname
- the node requires a password

### Comparison table

| Method | Best for | Internet needed | Works across different networks | What you enter |
|---|---|---:|---:|---|
| Hub/Portal | everyday automatic discovery | Yes | Yes | nothing after setup |
| Pair Code | adding a new device | Yes | Yes | short code |
| Share Link | advanced sharing | Usually | Yes | `clipshot://...` |
| Local Network Scan | same LAN | No | No | just click Scan |
| Manual Address | expert/manual setup | No | Sometimes | address + optional password |

## Subscription: Pro vs Lite

### Lite (free): $0/mo

Lite is the free plan.

What you get:
- up to **3 devices**
- local network sync
- basic clipboard history
- auto-update

What you don't get:
- no relay transport

What this looks like in the UI:
- sidebar badge says **Lite**
- Settings shows limited peer count and relay disabled
- extra devices may appear limited and offline
- a limited peer card can show **Activate** so you can swap it in

### Pro: $5/mo

Pro removes the free device cap and enables relay transport.

What you get:
- **unlimited** devices
- relay transport (sync anywhere)
- priority support
- everything in Lite

### Lifetime Pro: $100 one-time

Pay once, sync forever. All Pro features with no recurring payments.

What you get:
- everything in Pro
- no recurring payments
- lifetime updates
- all future Pro features included

Note: "Lifetime" refers to the lifetime of the product. See [Terms of Service](https://clipshot.cc/terms) for details.

### Grace period

If your Pro subscription was recently confirmed by Clipshot Portal, Clipshot keeps Pro features for a **14-day grace period** even if the hub is temporarily unreachable.

## Troubleshooting

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

## Data & Config Paths

### Settings: `~/.config/clipshot/settings.toml`

This is the main settings file on Linux and in the installer-based setup.

On some platforms, Clipshot may use the OS-equivalent config directory for the same file.

### Sync files: `~/.clipshot/sync/`

This folder stores synced files and clipboard assets.

### Peers: `~/.config/clipshot/peers.toml`

This file stores known/saved peers.

### HMAC secret: `~/.config/clipshot/hmac_secret`

This secret is used to validate Clipshot share links.

## Clipboard Toggle Hotkey

Default: **none** (not set). The input placeholder suggests `ctrl+a` as a starting point. Toggles clipboard between image/file content and the sync file path. Change in Settings → General → Clipboard Toggle Hotkey.

## Using Clipshot with VPN

If your devices are on the same Tailscale, WireGuard, or ZeroTier network, Clipshot connects through it automatically — no relay needed, even on the free plan.

## Multiple Groups

If your account has more than one group, the setup page shows a group picker. Choose which group the new device should join.
