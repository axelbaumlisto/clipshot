# Frequently Asked Questions

## How does Clipshot work?
Clipshot creates peer-to-peer encrypted connections between your devices using the iroh protocol (QUIC). When you copy something on one device, it's instantly sent to all connected peers.

## Is my data sent to any server?
No. Clipshot uses direct P2P connections. Your clipboard data never passes through our servers. The hub portal is only used for device discovery and pairing — it never sees your clipboard content.

## What platforms are supported?
- macOS (Intel and Apple Silicon)
- Linux (x64, including headless servers and WSL)
- Windows (x64)

## What's the difference between Lite and Pro?
- **Lite (Free)**: Up to 2 devices, local network sync
- **Pro ($5/mo)**: Unlimited devices, relay transport for cross-network sync

## How do I pair devices?
1. Open Clipshot on both devices
2. On one device, generate a pair code
3. Enter the code on the other device
4. Done — devices sync automatically

## Can I use Clipshot on a headless server?
Yes! Run `clipshot daemon --port 19231 --http-port 15282` for headless mode with HTTP API.

## How do I install via command line?
```bash
curl -fsSL https://clipshot.cc/install.sh | bash -s -- --code=WORD-WORD-00
```
