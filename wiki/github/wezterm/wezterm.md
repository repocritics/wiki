# wezterm/wezterm

> A GPU-accelerated cross-platform terminal emulator and multiplexer, configured in Lua and written in Rust.

[GitHub repo](https://github.com/wezterm/wezterm) ·
[Official website](https://wezterm.org/) ·
[License: MIT](https://github.com/wezterm/wezterm/blob/main/LICENSE.md)

## Overview

WezTerm is a terminal emulator written by Wez Furlong, started in 2018[^1]. It
occupies the same niche as Alacritty and kitty — GPU-accelerated rendering of a
modern terminal — but bundles two things those tools historically kept separate
or omitted: a full **multiplexer** (tabs, panes, and detachable sessions, in the
spirit of tmux) and a **Lua configuration language** in place of a static config
file[^2]. The result is one of the most feature-dense terminals available, at the
cost of being one of the larger and more complex ones.

The defining tension is scope. Most terminals pick a lane: Alacritty is
deliberately minimal and delegates tabs/sessions to tmux; kitty ships its own
protocols and kittens. WezTerm tries to be the terminal, the multiplexer, and the
SSH client at once, and to make all of it programmable in Lua. That breadth is the
reason people adopt it (one binary, one config, works the same on macOS, Linux,
Windows, and FreeBSD) and also the reason its config surface, release model, and
GPU behavior are more to reason about than a single-purpose emulator's.

It is a spare-time project maintained primarily by one author, which is worth
stating plainly: the README says so directly[^3], and it shapes the release
cadence and support expectations described below.

## Getting Started

Install from the official builds (Homebrew, apt/dnf repos, Windows installer,
or a direct download) — see the installation page[^4]:

```bash
# macOS
brew install --cask wezterm

# Windows (winget)
winget install wez.wezterm
```

Configuration lives in `~/.config/wezterm/wezterm.lua` (or `wezterm.lua` next to
the executable on Windows). It is real Lua, evaluated at startup and on change:

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()

config.color_scheme = 'Batman'
config.font = wezterm.font 'JetBrains Mono'
config.font_size = 13.0
config.hide_tab_bar_if_only_one_tab = true

-- Lua means config is computed, not just declared:
config.window_background_opacity =
  wezterm.target_triple:find 'darwin' and 0.92 or 1.0

return config
```

Config reloads automatically on save; a syntax or runtime error surfaces in a
config-error overlay instead of crashing the terminal.

## Architecture / How It Works

WezTerm is a Rust workspace split into reusable crates rather than a monolith. The
lowest layer is **termwiz**, a standalone terminal library (cell model, escape
sequence parsing, line editing, terminal capabilities) that is published
separately and usable outside WezTerm[^5]. On top of it sit the terminal-state
crate, the **mux** layer (the multiplexer: panes, tabs, windows, and the
client/server split), and **wezterm-gui**, the front-end that owns the window and
the renderer.

Rendering is GPU-driven. Glyphs are shaped with HarfBuzz and rasterized with
FreeType (or the platform font system), packed into a texture atlas, and drawn via
a GPU backend — WebGPU (wgpu) or OpenGL, with a software fallback path when no
usable GPU is present. Ligatures, color emoji, and complex-script shaping fall out
of the HarfBuzz pipeline rather than being bolted on.

The multiplexer is the part that most distinguishes the architecture. WezTerm
separates a **mux server** (which owns the PTYs and terminal state) from the GUI
**client**. A "domain" is a place panes can live: the local machine, an SSH host,
a TLS-secured remote mux, or a Unix-socket-attached local server. Because the
server holds state independently of the GUI, you can detach and reattach the way
you would with tmux — but you can also attach a native GPU-rendered window to a
remote session instead of tunneling a text protocol, which is the feature people
adopt WezTerm's multiplexer specifically for.

Lua is not a thin config veneer. The `wezterm` module exposes event hooks
(`format-tab-title`, key/mouse assignment callbacks, `update-status`), so the
config file is effectively a small program that can query the environment,
generate key tables, and render status bars. Images use sixel, the iTerm2
protocol, and the kitty graphics protocol.

## Production Notes

**Versioning is date-based, not semver.** Releases look like
`20240203-110809-5046fc22` (date-time-commit)[^6]. There is no "1.0"; the mental
model is a long-lived nightly plus infrequent tagged stable builds. Practical
consequence: a large amount of documented functionality lands in nightly first,
and a stable build can sit unchanged for many months while nightlies move. Read
the changelog and check whether a feature you want requires nightly before filing
a bug against stable.

**GPU/front-end is the most common source of environment-specific pain.** WebGPU
vs OpenGL selection, Wayland vs X11, fractional/multi-monitor scaling, and NVIDIA
proprietary drivers have historically produced blank windows, wrong DPI, or
tearing on specific setups. `front_end` and `webgpu_*` options exist precisely to
switch backends when the default misbehaves; a "blank terminal" report is usually
a backend selection issue, not a terminal-logic bug.

**One-maintainer support model.** The README states this is a spare-time
project[^3]. Issue volume is high (well over a thousand open at any time), and
response is best-effort. This is not a knock — it is the operating reality to plan
around for a tool in your critical path.

**Resource footprint.** Being a GPU app with a glyph atlas and an embedded Lua
runtime, WezTerm's baseline memory and binary size are larger than Alacritty's.
For most desktops this is a non-issue; on constrained remote/VM environments it is
worth measuring rather than assuming.

**Multiplexer is not a drop-in tmux.** Keybindings, session-persistence
semantics, and the muscle memory differ. Teams standardized on tmux scripting or
tmux-resurrect-style persistence should expect to re-learn and re-tool rather than
alias their way across.

## When to Use / When Not

**Use when:**
- You want one program to be terminal + multiplexer + SSH client, identical across
  macOS, Linux, Windows, and FreeBSD.
- You want programmable config (dynamic status bars, computed keybindings) rather
  than a static file.
- You want to attach a GPU-rendered window to a remote multiplexer session.
- You want ligatures, color emoji, and image protocols working out of the box.

**Avoid when:**
- You want a minimal, single-purpose emulator and already run tmux — Alacritty is
  smaller and simpler for that.
- You need a specific tagged stable with tight change control; the date-based
  nightly-forward model may frustrate you.
- You are on a GPU/Wayland/driver combination known to be finicky and don't want
  to tune backends.
- You need vendor-backed SLAs or fast issue turnaround for a critical dependency.

## Alternatives

- alacritty/alacritty — minimal GPU terminal, TOML config, no tabs/multiplexer;
  use it when you want a small fast emulator and drive sessions with tmux.
- kovidgoyal/kitty — GPU terminal with its own graphics protocol and "kittens";
  use it when you're invested in the kitty ecosystem and image workflows.
- ghostty-org/ghostty — newer GPU terminal focused on native platform integration;
  use it when native macOS/GTK feel matters more than a built-in multiplexer.
- tmux/tmux — pure multiplexer, terminal-agnostic; use it when you want the
  battle-tested session-persistence standard under any emulator.
- contour-terminal/contour — cross-platform GPU emulator with strong protocol
  support; use it as an alternative feature-rich single emulator.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial repo | 2018-02 | Project started; Rust terminal + multiplexer[^1]. |
| Date-based releases | 2019+ | Adopted `YYYYMMDD-HHMMSS-<commit>` version scheme[^6]. |
| 20220319-142410 | 2022-03 | Multiplexer, Lua config, and image protocols matured. |
| 20230326-111934 | 2023-03 | Continued stable line; WebGPU front-end work. |
| 20240203-110809 | 2024-02 | Long-standing tagged stable; nightlies lead thereafter[^6]. |

## References

[^1]: WezTerm repository, created 2018-02-07. https://github.com/wezterm/wezterm
[^2]: WezTerm features overview. https://wezterm.org/features.html
[^3]: WezTerm README, "Getting help" — "This is a spare time project." https://github.com/wezterm/wezterm/blob/main/README.md
[^4]: WezTerm installation guide. https://wezterm.org/installation
[^5]: termwiz crate on crates.io. https://crates.io/crates/termwiz
[^6]: WezTerm changelog and version scheme. https://wezterm.org/changelog.html

## Tags

rust, terminal-emulator, terminal-multiplexer, gpu-accelerated, cli, cross-platform, lua-config, ssh, developer-tools, tui
