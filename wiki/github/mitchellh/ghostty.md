# mitchellh/ghostty

> A fast, feature-rich, cross-platform terminal emulator with native UI and GPU acceleration.

[GitHub repo](https://github.com/mitchellh/ghostty) ·
[Official website](https://ghostty.org) ·
[License: MIT](https://github.com/mitchellh/ghostty/blob/main/LICENSE)

## Overview

Ghostty is a terminal emulator written in **Zig** by Mitchell Hashimoto (co-founder of HashiCorp, author of Vagrant and the original Terraform)[^1]. It targets macOS (AppKit), Linux (GTK 4), with a Windows port in progress. The renderer is GPU-accelerated (Metal on macOS, OpenGL on Linux), and the UI uses native widgets on each platform rather than a shared cross-platform toolkit.

The 1.0 release in December 2024 was the first public release after several years of private beta[^2]. Since then 1.1 (2025-04) and ongoing 1.x point releases have focused on stability, theming, kitty graphics protocol support, configuration ergonomics, and Windows readiness.

Ghostty's positioning is "fast and feature-complete with native UI everywhere". This contrasts with:
- **Alacritty** — speed-focused, no tabs/splits, GLFW-based UI, minimal config.
- **WezTerm** — Lua-scriptable, GPU-accelerated, cross-platform but with non-native UI on macOS.
- **iTerm2** — feature-rich, mature, but macOS-only and not GPU-accelerated end-to-end.
- **kitty** — fast, cross-platform, opinionated kitty graphics protocol pioneer, non-native UI.

The choice of Zig as the implementation language is notable. Mitchell has been a prominent Zig advocate; Ghostty is one of the largest public Zig codebases.

## Getting Started

macOS (Homebrew):

```bash
brew install --cask ghostty
```

Linux (Flatpak, snap, distro packages):

```bash
flatpak install flathub com.mitchellh.ghostty
# or distro-specific packages
```

Build from source (Zig 0.13+):

```bash
git clone https://github.com/mitchellh/ghostty
cd ghostty
zig build -Doptimize=ReleaseFast
```

Configuration lives at `~/.config/ghostty/config`:

```
theme = catppuccin-mocha
font-family = JetBrainsMono Nerd Font
font-size = 14
window-decoration = false
background-opacity = 0.92
keybind = ctrl+shift+t=new_tab
```

Run with custom config:

```bash
ghostty --config-file=~/myconfig
```

## Architecture / How It Works

Ghostty is structured as a layered codebase[^3]:

1. **`libghostty`** — the core terminal engine, written in Zig. Parses ANSI/VT escape sequences, maintains screen state, exposes a buffer/cell API. Reusable as a library; in principle other front-ends could embed it.
2. **`termio`** — terminal I/O (pty management, child process spawning).
3. **Renderer** — Metal (macOS) and OpenGL (Linux) backends, both rendering text via signed distance field fonts.
4. **Platform UI** — AppKit on macOS, GTK 4 on Linux. Tabs, splits, menus, window chrome are platform-native.

The renderer uses **GPU instanced rendering** for text. Each glyph is a textured quad drawn via instancing; cell attributes (color, bold, italic) are vertex attributes. This is competitive in throughput with Alacritty and kitty.

**Configuration syntax** is a custom flat key-value format, deliberately simpler than TOML/YAML. Color themes are first-class, with a default set including Catppuccin, Solarized, Nord, Gruvbox, Tokyo Night.

**Keybindings** are declared as `keybind = <chord>=<action>`. Actions include `new_tab`, `new_split`, `next_tab`, `goto_split`, `clear_screen`, `decrease_font_size`, etc. Multi-key chords (`ctrl+a>c`) are supported.

**Terminfo.** Ghostty ships its own `terminfo` entry (`ghostty` or `xterm-ghostty`). On SSH to remote hosts that don't have it, fall back to `TERM=xterm-256color`. The `ghostty +install-terminfo` command installs into the local user terminfo database.

**Kitty graphics protocol.** Ghostty 1.1+ supports the kitty graphics protocol[^4], enabling tools like `kitty +kitten icat`, `chafa`, and `nvim`'s image.nvim plugin to display real bitmaps in the terminal.

## Production Notes

**Performance vs Alacritty.** In simple throughput benchmarks (vtebench), Ghostty and Alacritty are within margin of error. Ghostty wins on UI responsiveness (native widgets, fewer custom event loop quirks); Alacritty wins on minimum memory footprint.

**Configuration ergonomics.** The config language is simple but the surface area is large (>200 options as of 1.1). The `ghostty +show-config --default --docs` command dumps the full annotated config — treat it as the canonical reference.

**Themes.** The shipped themes are good. Custom themes go in `~/.config/ghostty/themes/<name>`. Theme switching at runtime via keybind action `set_theme`.

**Shell integration.** Ghostty supports OSC 133 prompt marking, automatic shell integration injection for bash/zsh/fish. The integration enables features like "jump to previous prompt" and clickable URLs that respect the cursor position. Bash users may need to opt in manually.

**Window decoration on Linux.** The Linux GTK build has had several iterations on window decorations (CSD vs SSD). Tiling WM users typically set `window-decoration = false`. Wayland support is functional but less polished than X11 historically; this is improving rapidly.

**Windows.** Officially still in development as of 1.1. Community Windows builds exist. Targeting first-class Windows support in a future 1.x release.

**Crash recovery.** Ghostty stores config and minimal state in `~/.config/ghostty`. There is no persistent terminal scrollback across restarts (a deliberate choice). For session restoration, use `tmux` / `zellij` inside Ghostty.

**Resource use.** Idle Ghostty uses 50–100 MB RAM per window (typical for GPU-accelerated terminals with text caches). Heavy SGR (color) output can spike GPU usage on integrated graphics; modern dedicated GPUs are unbothered.

## When to Use / When Not

**Use when:**
- You're on macOS or Linux and want a native, fast, feature-rich terminal.
- You want kitty graphics protocol + ligatures + GPU rendering + native UI in one tool.
- You're willing to be on a 1.x release of relatively new software.
- You value Mitchell's design taste and the project's stability commitment.

**Avoid when:**
- You're on Windows and need rock-solid stability today.
- You need a Lua-scriptable terminal (use WezTerm).
- You need the absolute minimum memory footprint (Alacritty wins by ~30 MB).
- You depend on tmux features that the terminal itself provides (tabs/splits) and want one-tool-does-it-all (WezTerm has more overlap there).

## Alternatives

- **Alacritty** — minimal, fast, no tabs/splits, GLFW-based.
- **WezTerm** — Lua config, cross-platform, more features but non-native UI.
- **kitty** — fast, cross-platform, kitty graphics protocol creator, non-native UI.
- **iTerm2** — macOS-only, feature-rich, mature, not GPU-accelerated end-to-end.
- **Warp** — Rust-based, AI-features-first, partially proprietary.

## History

| Version | Date | Notes |
|---------|------|-------|
| Private beta | 2022–2024 | Closed beta, invite-only. |
| 1.0 | 2024-12-26 | First public release[^2]. |
| 1.0.1 | 2025-01 | Stability fixes. |
| 1.1 | 2025-04 | Kitty graphics protocol, theme improvements. |
| 1.x | 2025 | Ongoing Windows work, config refinement. |

## References

[^1]: Mitchell Hashimoto, personal blog. https://mitchellh.com/ghostty
[^2]: Mitchell Hashimoto, "Ghostty 1.0" — 2024-12. https://mitchellh.com/writing/ghostty-1-0-reflection
[^3]: Ghostty docs, "Architecture". https://ghostty.org/docs
[^4]: kitty docs, "Terminal graphics protocol". https://sw.kovidgoyal.net/kitty/graphics-protocol/
