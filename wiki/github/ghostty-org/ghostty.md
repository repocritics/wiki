# ghostty-org/ghostty

A fast, feature-rich, cross-platform terminal emulator — Zig-implemented with GPU acceleration and platform-native UI. Authored by Mitchell Hashimoto (HashiCorp founder).

## What it is

A terminal emulator that compiles to native binaries on macOS, Linux, and Windows using GPU-accelerated rendering and platform-native UI (real NSWindow on macOS, Win32 on Windows, GTK on Linux). Written in Zig — chosen for low-level control + cross-platform binary distribution. Originally a personal project by Mitchell Hashimoto; transitioned to `ghostty-org` org for community stewardship.

## Key features

- GPU-accelerated rendering with attention to throughput and latency.
- Native UI per platform (no Electron, no abstraction layer).
- Cross-platform: macOS, Linux, Windows.
- True color, ligatures, font customization.
- Fast startup, low memory footprint.
- Configuration via plain text file.
- MIT-licensed.

## Tech stack

- Zig primary — chosen for the GPU + cross-platform binary story.
- Platform-native UI bindings.

## When to reach for it

- You want a high-performance terminal with native platform integration.
- You're a power user benchmarking terminal latency for daily use.
- You're interested in Zig-implemented systems software.

## When *not* to reach for it

- You need bleeding-edge SSH / multi-protocol features bundled — Ghostty is a terminal emulator, not a swiss-army knife like Tabby.
- You want extensive plugin ecosystem — Ghostty is intentionally lean.

## Maturity signal

56k stars, 2.8k forks, MIT, actively maintained under the ghostty-org organization (formerly `mitchellh/ghostty` — repo transferred to the org as community stewardship grew). The Zig choice + Mitchell Hashimoto's reputation drove early adoption. Open-issues count of 243 is moderate for active development.

## Alternatives

- Alacritty — Rust-based GPU-accelerated terminal.
- WezTerm — Lua-configurable GPU terminal with extensive features.
- Kitty — fast GPU terminal with multiplexing.
- iTerm2 (macOS), Windows Terminal — platform-specific.

## Tags

zig, terminal, gpu, cross-platform, mit-license, developer-tools, native
