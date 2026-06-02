# 2dust/v2rayN

A C#-based desktop GUI client for the V2Ray / Xray / sing-box proxy ecosystem on Windows, Linux, and macOS.

## What it is

A Windows-first (now cross-platform) GUI frontend for the V2Ray family of proxy tools. Bundles the proxy engine binaries (Xray, sing-box, v2fly) and adds a desktop UI for managing servers, routing rules, subscription imports, and traffic statistics. Supports the full V2Ray-protocol family: VMess, VLESS, Trojan, Shadowsocks, SOCKS5, plus XTLS transport. GPL-3.0 licensed.

## Key features

- Cross-platform desktop GUI (Windows, Linux, macOS) for V2Ray / Xray / sing-box engines.
- Protocol coverage: VMess, VLESS, Trojan, Shadowsocks, SOCKS5, XTLS.
- Subscription-link import with auto-update.
- Routing rules editor with per-app proxy support on Windows.
- Server speed-test, latency probing, and traffic statistics.
- Multi-language UI.
- GPL-3.0 licensed.

## Tech stack

- C# / .NET primary on the UI side.
- Bundles Xray-core, sing-box, and v2fly binaries.
- Distributed via GitHub Releases (Windows MSI + portable, Linux DEB/RPM, macOS DMG).

## When to reach for it

- You're a V2Ray / Xray user on desktop and want a polished GUI instead of YAML config.
- You're managing many proxy servers and want subscription-based bulk import.
- You need per-app routing rules on Windows.

## When *not* to reach for it

- You're on mobile — different clients (Shadowrocket on iOS, v2rayNG on Android) fit better.
- You want a CLI-only setup — the engine binaries (xray, sing-box) run headless.
- You're allergic to GPL-3.0 — copyleft applies to derivative GUIs.

## Maturity signal

107k stars, 15k forks, GPL-3.0, last push the day this page was generated. 6-year-old project under the 2dust org with active development. Open-issues count of 14 is exceptionally low. The proxy-tooling ecosystem operates in a politically charged space (network-censorship circumvention) where active maintenance + stable release cadence matter for security patches.

## Alternatives

- `clash-verge-rev/clash-verge-rev` — Tauri-based alternative for the Clash/Mihomo ecosystem.
- v2rayA — web-UI alternative for the same V2Ray engine.
- Nekoray / NekoBox — alternative cross-platform clients.

## Notes

The V2Ray / Xray / sing-box engine choice is the underlying decision; v2rayN is a frontend for whichever you pick. Anyone in the proxy-tooling space should pin to known-good binary versions of the engines since upstream changes occasionally break compatibility. GPL-3.0 + active maintenance + Windows-first stewardship make this the dominant desktop choice for V2Ray-family users on Windows.

## Tags

c-sharp, proxy, networking, v2ray, xray, sing-box, windows, gpl, desktop
