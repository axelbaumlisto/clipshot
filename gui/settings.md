---
layout: default
title: Settings
nav_order: 4
parent: Application
---
![Settings page](../docs/images/settings.png)

The Settings page is split into collapsible cards. Clipshot remembers which sections you keep open or closed.

<img src="../docs/images/settings-annotated.png" alt="Settings page annotated" class="img-wide">

The annotated view highlights: ① **Connection** card — Hub Portal URL, Group Token, subscription info. ② **General** card — theme, startup, notifications, hotkey. ③ **Save Changes** / **Reset to Defaults** buttons at the bottom.

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
- the default sync toggle hotkey is **Ctrl+B**
- changing hotkeys requires a restart
- the clipboard toggle hotkey is separate from pause/resume sync

### Sync (catchup_limit, max_file_size)

The **Sync** card includes:
- **Max file size (MB)**
- **Catch-up sync limit**

What they mean:
- **Max file size** limits what Clipshot will send
- **Catch-up sync limit** controls how many recent clipboard entries a reconnecting device can fetch

If you set a very large file limit, Clipshot warns that large files may time out.

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
