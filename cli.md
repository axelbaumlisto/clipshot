---
layout: default
title: CLI mode
nav_order: 5
---
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

### systemd / launchd auto-start

Install Clipshot as a background service:

```bash
clipshot service install --port 19231 --http-port 18080
```

Manage it with:

```bash
clipshot service status
clipshot service logs
clipshot service uninstall
```

Notes:
- Linux uses a **systemd user service**
- macOS uses a **launchd agent**
- the one-line installer sets this up automatically for you
