# rustdesk/rustdesk

A self-hosted, open-source remote desktop alternative to TeamViewer — Rust-implemented client + relay, multi-platform, AGPL-3.0.

## What it is

A Rust-based remote-desktop application that lets users control one machine from another over the internet. Architecturally: client app (Flutter + Rust) on every device, optional self-hosted relay server (RustDesk Server) for sessions that can't establish direct peer-to-peer connections, public relay infrastructure as a fallback. The pitch is TeamViewer's UX without TeamViewer's pricing or telemetry — and entirely OSS. AGPL-3.0 licensed.

## Key features

- Cross-platform client: Windows, macOS, Linux, Android, iOS.
- P2P first; relay fallback when direct connection fails.
- Self-hostable relay server (separate `rustdesk-server` repo) for organizations that don't trust the public relay.
- File transfer, clipboard sync, multi-monitor support, audio forwarding.
- TCP, UDP, and Wayland-display support.
- Flatpak distribution on Linux.
- AGPL-3.0 licensed.

## Tech stack

- Rust primary on the runtime side (encoding, networking, encryption).
- Flutter for the UI layer (Dart on the frontend).
- Audio/video via standard codec libraries (FFmpeg, libvpx, etc.).

## When to reach for it

- You want a free, OSS alternative to TeamViewer / AnyDesk for personal or internal-org remote support.
- You're privacy-sensitive and want a self-hosted relay rather than relying on a commercial vendor's servers.
- You need cross-platform — RustDesk's mobile (iOS + Android) support is unusual for a self-hostable remote-desktop tool.

## When *not* to reach for it

- You need enterprise SLAs + support contracts — TeamViewer / AnyDesk / Splashtop offer them; RustDesk is community-supported.
- You're allergic to AGPL — the copyleft pulls SaaS-style redistribution into source-release.
- You need certified-compliance attestations (HIPAA, FedRAMP, etc.) — community-OSS doesn't supply these without paid audit.

## Maturity signal

115k stars, 17k forks, AGPL-3.0, last push the day this page was generated. 5-year-old project with very active development. Open-issues count of 108 is unusually low for the platform breadth (5 OSes) — the maintainers triage aggressively. The AGPL choice + self-hostable relay is the typical "commercial-OSS hybrid" pattern; RustDesk Inc. operates the hosted offering, while the client is OSS.

## Alternatives

- TeamViewer / AnyDesk / Splashtop — use when you want commercial vendor support and SLAs.
- Apache Guacamole — use when you want browser-based remote desktop (HTML5).
- VNC / RDP directly — use when you have direct network access and don't need a relay.
- Parsec — use when you specifically want low-latency gaming-grade remote desktop.

## Notes

The "alternative to TeamViewer" framing is direct competition with a specific commercial product; that drives both the project's high adoption and TeamViewer's occasional legal pushback against the brand-positioning. AGPL-3.0 is the right license for this niche — it protects the self-hosting use case while making fork-and-resell legally meaningful. Self-hosted relay is the most-distinctive operational property; organizations with strict network policies often prefer it over the public relay.

## Tags

rust, flutter, remote-desktop, p2p, self-hosted, cross-platform, agpl, networking, vnc, rdp
