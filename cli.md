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

### Account setup via CLI

Set up a new account without a pair code. This creates a session on the portal, opens your browser for login or registration (Google OAuth supported), and polls until you authorize the device. The group token is saved automatically.

```bash
clipshot setup
```

If you already have a group token, `clipshot setup` tells you so and exits. To switch groups, use `clipshot pair CODE` instead.

### Useful commands

Generate a share link for this node:

```bash
clipshot share-uri
clipshot share-uri --port 19231 --ttl 3600 --name mybox
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
clipshot history --filter images --limit 10
```

Filters: `all`, `files`, `images`, `text`. Default limit is `20`.

Check for or install updates:

```bash
clipshot update
clipshot update --check
```

`clipshot update` checks clipshot.cc for a newer release matching your platform. With `--check` it only reports availability without installing.

Other useful CLI commands:

```bash
clipshot status
clipshot pause
clipshot resume
clipshot toggle
clipshot doctor
clipshot list-peers
clipshot add-peer alice 192.168.1.5:19231
clipshot remove-peer alice
clipshot retry-transfer /path/to/file
```

### Settings management

View or change settings from the command line:

```bash
clipshot config show
clipshot config get listen_port
clipshot config set max_file_size_mb 100
```

Manage saved peers:

```bash
clipshot peers list
clipshot peers add bob 10.0.0.5:19231
clipshot peers remove 10.0.0.5:19231
```

### Shell completions

Generate completion scripts for your shell:

```bash
clipshot completions bash > ~/.local/share/bash-completion/completions/clipshot
clipshot completions zsh > ~/.zfunc/_clipshot
clipshot completions fish > ~/.config/fish/completions/clipshot.fish
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
