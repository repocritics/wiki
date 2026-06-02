# Eugeny/tabby

A modern terminal emulator with built-in SSH client, serial console, and Telnet — Electron-based, cross-platform, plugin-extensible.

## What it is

A terminal emulator that targets the gap between traditional terminals (iTerm2, Windows Terminal) and "I need an SSH/serial/Telnet client" tools (PuTTY, MobaXterm). Built on Electron + TypeScript for cross-platform consistency. Bundles SFTP, port forwarding, and a plugin system. Distinct from `huacnlee/tabby` and the unrelated `TabbyML/tabby` (an AI coding agent).

## Key features

- Multi-protocol terminal: local shell, SSH, serial, Telnet, port forwarding.
- SSH client with credential vault, jump hosts, mosh support.
- SFTP integration alongside SSH sessions.
- Tab management, split panes, tab grouping.
- Theme + plugin system; many community plugins.
- Cross-platform (Windows, macOS, Linux).
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Electron for cross-platform packaging.

## When to reach for it

- You want one terminal app that handles local + SSH + serial + Telnet without separate tools.
- You're on Windows and want MobaXterm-class features in an OSS package.

## When *not* to reach for it

- You want a lightweight terminal — Tabby is Electron, ships ~150MB.
- You want a vendor-supported terminal — iTerm2, Windows Terminal, Warp have larger teams.

## Maturity signal

72k stars, 4k forks, MIT, actively maintained. Open-issues count of 2,697 is high but tracks the breadth of platform-specific + protocol-specific reports.

## Alternatives

- Warp — modern, AI-augmented terminal (commercial).
- iTerm2 (macOS), Windows Terminal — platform-native.
- Hyper — Electron-based but more minimal.

## Tags

typescript, electron, terminal, ssh-client, cross-platform, mit-license, developer-tools
