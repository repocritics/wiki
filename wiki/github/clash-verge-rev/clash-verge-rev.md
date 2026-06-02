# clash-verge-rev/clash-verge-rev

A modern Tauri-based GUI client for the Clash / Mihomo proxy ecosystem — desktop frontend for routing network traffic through proxy configurations.

## What it is

A cross-platform desktop client for Clash (and the Mihomo successor) that wraps the proxy engine in a GUI. Built with Tauri (Rust backend + web frontend) for native binaries on Windows, macOS, and Linux. The "rev" suffix marks this as the revived / community-maintained fork after the original Clash Verge stopped releasing — the rev fork has become the de facto active maintenance lineage. GPL-3.0 licensed.

## Key features

- Cross-platform desktop GUI (Windows, macOS, Linux) via Tauri.
- Clash / Mihomo configuration editing, rule management, and traffic visualization.
- Subscription-based config import with auto-update.
- TUN mode, system-proxy mode, and per-app proxy on supported platforms.
- Active fork lineage tracking Mihomo / Clash.Meta core upgrades.
- GPL-3.0 licensed.

## Tech stack

- TypeScript primary on the renderer.
- Tauri (Rust) on the backend / native bridge.
- Embeds Mihomo / Clash binaries for the proxy engine itself.

## When to reach for it

- You want a desktop GUI for managing Clash / Mihomo proxy configurations across multiple platforms.
- You're a Mihomo user wanting an actively-maintained frontend after the original Clash Verge slowed.
- You need TUN-mode or per-app proxy routing through a managed UI.

## When *not* to reach for it

- You're on iOS / Android — this is desktop-only; mobile users want Stash, Surfboard, or Clash for Android instead.
- You're allergic to GPL-3.0 — the copyleft is real and matters if you embed Clash Verge into another product.
- You want a CLI-only setup — the proxy engine binaries (Mihomo, Clash) run headless if that's the goal.

## Maturity signal

122k stars, 9k forks, GPL-3.0, last push the day this page was generated. 2-year-old fork with active development cadence. The `dev` default branch is the operational signal — releases come from tagged commits, not from `main`. Open-issues count of 391 tracks per-platform networking quirks. Community-maintained but the lineage is stable.

## Alternatives

- Clash for Windows (CFW) — historical desktop frontend; effectively retired upstream but still installed widely.
- Stash, Shadowrocket — iOS clients (different platform).
- FlClash, ClashX Pro — alternative cross-platform frontends with different UX choices.
- Direct Mihomo CLI — use when you don't need a GUI at all.

## Notes

The proxy-tooling ecosystem operates in a politically-charged space (network-censorship circumvention is a real use case). Anyone evaluating this should know the dependency chain (Clash → Mihomo → Clash Verge → clash-verge-rev) and the maintainer lineage history. GPL-3.0 means commercial redistributions must release their changes; that's intentional in this niche.

## Tags

tauri, typescript, rust, desktop, gui, proxy, networking, clash, mihomo, cross-platform, gpl
