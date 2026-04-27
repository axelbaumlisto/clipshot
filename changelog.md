---
layout: default
title: Changelog
nav_order: 10
---
## v0.5.1

### Added
- **Device Auth Flow**: `curl -fsSL clipshot.cc/install.sh | bash` works without `--code`; opens browser for registration
- **Google OAuth** on portal login/register/setup pages
- **CLI `clipshot setup`** command for browser-based account creation
- **CLI `clipshot update`** command for checking and installing updates
- **GUI WelcomeScreen** now has a **Create Account** button for browser-based registration
- **Tray menu**: added **Copy Last Synced** item
- **Permissions commands**: `check_permissions`, `open_permission_settings`, `restart_app_for_permissions`
- **HTTP route**: `GET /api/permissions` for permission status
- **Group picker** for users with multiple groups on portal

### Changed
- `enable_iroh` serde default fixed: was `false`, now correctly defaults to `true`
- `clipboard_hotkey` default changed from `a` to `ctrl+a`
- Install script no longer requires `--code` parameter

### Fixed
- `hub_connected` fix for browser mode (`drain_hub_events`)
- Portal `relay_enabled` auto-update on subscription change

## v0.5.0

### Added
- Browser-mode and desktop UI parity improvements across Overview, Peers, History, and Settings
- Compact header controls for sync state and peer presence
- Storage retention settings for sync files: file count, max age, and max total size
- Persistent outbox state so reconnect catch-up does not resend already-delivered items
- GitHub Pages documentation site with split guides, search, and sidebar navigation

### Changed
- Documentation reorganized into task-focused pages instead of one long scrolling guide
- Screenshots now use tighter presentation and consistent sizing rules in the site theme
- Theme styling updated to match the main Clipshot website palette

### Fixed
- Catch-up sync duplicates after restart by persisting delivery state
- Hotkey deadlock and pause/resume responsiveness issues in the GUI
- Sync state desync between transfer activity and visible UI indicators
- macOS permission/update edge cases around Input Monitoring and app identity

## Earlier releases

# Changelog

All notable changes to this project will be documented in this file.

## [0.2.2] - 2026-02-15

### Added
- Configurable transfer timeouts for large file transfers
  - `transfer_timeout_per_mb_ms` - timeout per MB (default 500ms)
  - `transfer_timeout_base_ms` - base timeout (default 5000ms)
  - `chunk_timeout_secs` - chunk-level timeout (default 120s)
- GUI: Advanced Transfer Settings section in Sync Settings
- HTTP API: New timeout fields in POST /api/settings
- CLI: `clipshot config transfer_timeout_per_mb_ms 1000`

### Fixed
- 100MB stress test now passes reliably (205s transfer time)

## [0.2.1] - 2026-02-15

### Added
- PeerRegistry: Single source of truth for peer configuration
- GUI event loop for bidirectional clipboard sync
- Clipboard polling in GUI daemon

### Fixed
- GUI daemon receiving peer content
- Bidirectional sync (Mac <-> Docker)

## [0.2.0] - 2026-02-14

### Added
- Unified binary architecture (GUI + CLI in one)
- P2P v2 mesh networking with chunked transfer
- Tauri GUI with system tray
- Display auto-detection (Linux/macOS/Windows)
- Unified `clipshot://` URI format (GUI + CLI + HTTP API)

### Changed
- No feature flags - all functionality included by default
