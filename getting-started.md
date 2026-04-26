---
layout: default
title: Getting Started
nav_order: 2
---

# Getting Started

## Installation

### One-liner (recommended)

```bash
curl -fsSL https://clipshot.cc/install.sh | bash
```

### One-liner with pairing

```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=WORD-WORD-00
```

### macOS via Homebrew

```bash
brew install --cask clipshot
```

### Download manually

Visit [clipshot.cc/download](https://clipshot.cc/#download) for platform-specific binaries.

### Headless / Server

```bash
clipshot daemon --port 19231 --http-port 15282
```

## Pairing

### Using pair code

1. On Device A: open Clipshot → click **Pair device**
2. A pair code appears (e.g., `BLUE-FISH-42`)
3. On Device B: enter the code
4. Done — devices sync automatically

### Using share link

1. On Device A: **Settings → Share URI**
2. Copy the `clipshot://...` link
3. On Device B: **Pair device → Paste Link**

### Using local network

Both devices on the same WiFi? Clipshot finds them automatically via mDNS.

## First Sync

Once paired, simply **copy** anything on one device — text, image, or file — and it appears on all connected devices instantly.

Press **Ctrl+B** to toggle between the synced file path and the actual content in your clipboard.
