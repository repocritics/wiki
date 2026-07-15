# kovidgoyal/kitty

> A GPU-rendered terminal emulator that treats the terminal as a platform, shipping its own image and keyboard protocols rather than waiting for VT100 to grow.

[GitHub repo](https://github.com/kovidgoyal/kitty) ·
[Official website](https://sw.kovidgoyal.net/kitty/) ·
[License: GPL-3.0](https://github.com/kovidgoyal/kitty/blob/master/LICENSE)

## Overview

kitty is a cross-platform (Linux, macOS) terminal emulator written primarily
in C and Python, with a growing Go layer, by Kovid Goyal — the author of the
calibre e-book manager. The repository dates to 2016[^gh]. Its defining
decision is to render glyphs on the GPU via OpenGL rather than on the CPU,
which lets it push large scrollback and high-refresh output cheaply while
offloading compositing work off the main thread.

The larger thesis is that a terminal need not be a museum piece. kitty ships
protocol extensions that the traditional terminal stack never standardized: an
inline graphics protocol for displaying real images (not sixel hacks), a
comprehensive keyboard protocol that disambiguates key events the legacy
encoding cannot, and a structured remote-control interface over a socket.
Several of these — notably the keyboard protocol — have since been adopted by
other emulators, so kitty functions as a de facto protocol laboratory as much
as an application.

The tension is governance and coupling. kitty is a single-maintainer project
run with a firm hand: feature requests are frequently declined, bug reports
without a minimal reproducer are closed quickly, and design decisions are not
up for community vote[^issues]. That produces a coherent, fast product, but it
also means the roadmap is one person's, and its non-standard protocols only
help you where the rest of your toolchain (tmux, ssh, editors) speaks them.

## Getting Started

```bash
# Official installer (Linux/macOS) — writes to ~/.local, no root
curl -L https://sw.kovidgoyal.net/kitty/installer.sh | sh /dev/stdin

# Or a package manager
brew install --cask kitty        # macOS
# most Linux distros package it as `kitty` or `kitty-terminal`
```

```conf
# ~/.config/kitty/kitty.conf — a minimal, valid config
font_family        JetBrains Mono
font_size          13.0
background_opacity  0.95
enabled_layouts    splits,stack

# kitty has native tabs/windows/layouts; no tmux required for splits
map ctrl+shift+enter  new_window
map ctrl+shift+t      new_tab
```

```bash
# Display an image inline using the graphics protocol
kitty +kitten icat path/to/image.png
```

Reload config live with `ctrl+shift+f5` (or `SIGUSR1`); most options apply
without a restart, though a few (e.g. some font settings) need a relaunch.

## Architecture / How It Works

kitty is deliberately layered by language, each chosen for a job:

1. **C core** — the VT parser, the OpenGL renderer, and the glyph cache. Cells
   are rasterized once into a texture atlas and drawn as textured quads, so
   redrawing a full screen is a GPU operation rather than per-glyph CPU work.
2. **Python** — configuration, layout logic, session handling, and the older
   "kittens" (kitty's term for bundled sub-tools). Startup imports Python, so
   the binary embeds an interpreter.
3. **Go** — newer kittens and performance-sensitive helpers, most visibly the
   `ssh` kitten. Go compiles to a static helper that avoids shipping a Python
   runtime to the remote side.

Unlike tmux, kitty is not a client/server multiplexer by default: each window
is the process. It does, however, expose a **remote-control protocol** over a
unix socket (`kitty @ launch`, `kitty @ set-colors`, etc.), gated behind
`allow_remote_control`, which lets scripts drive windows, tabs, and layouts.
A "single instance" mode (`kitty --single-instance`) routes new invocations
into an existing process.

The **graphics protocol** transmits pixel data (PNG or raw RGBA, optionally via
shared memory or a temp file to avoid escaping megabytes through the pty) and
places it at cell coordinates, with z-index and animation. The **keyboard
protocol** uses progressive enhancement: an application queries support, then
opts into precise key-event reporting. Both are escape-sequence protocols, so
they degrade to no-ops on terminals that don't understand them — but any
program in between that mangles escape sequences (some tmux versions, some ssh
setups) breaks them.

kittens are the extension mechanism: `icat` (images), `diff` (side-by-side with
images), `hints` (URL/path selection), `unicode_input`, `themes`, and `ssh`.
They ship in-tree and are invoked as `kitty +kitten <name>`.

## Production Notes

**Remote terminfo is the number-one friction point.** kitty sets
`TERM=xterm-kitty`, whose terminfo entry is often absent on servers you ssh
into, producing "unknown terminal type" errors and broken key handling. The
`ssh` kitten works around this by copying the terminfo database to the remote
host on connect; plain `ssh` does not. Teams commonly either use the kitten,
pre-install the terminfo entry via `infocmp`/`tic`, or set `TERM=xterm-256color`
for remote sessions and accept reduced fidelity.

**It needs a real GPU/OpenGL.** kitty requires OpenGL 3.3+. Headless servers,
some VMs, nested/remote display setups, and older/software GL stacks can fail to
start or fall back poorly. This is the tradeoff for GPU rendering — it is not a
drop-in on machines where CPU-only emulators just work.

**GPL-3.0 has bundling consequences.** Unlike the permissive licenses common to
Rust-based competitors (Alacritty, Ghostty), redistributing kitty inside a
larger product carries copyleft obligations. This matters for anyone shipping a
terminal as part of a proprietary tool.

**The protocols only pay off end-to-end.** Inline images, precise keyboard
handling, and remote control assume your multiplexer, editor, and ssh path all
cooperate. Under a multiplexer that doesn't pass the protocols through, you lose
the features that justify choosing kitty in the first place.

**Maintainer posture is part of operating it.** Expect terse issue triage and a
strong bias toward "provide a reproducer or it's closed." This keeps quality
high but means edge-case bugs on unusual hardware may be resolved with "not
supported" rather than a fix[^issues]. Plan support expectations accordingly.

## When to Use / When Not

**Use when:**
- You live in the terminal and want native tabs, splits, and layouts without
  running tmux for local work.
- You want inline images (plotting, image previews, `icat`) or precise keyboard
  handling in a TUI/editor that speaks the kitty protocols.
- You render heavy output (logs, TUIs, fast scroll) and want GPU compositing.
- You want scriptable window management via the remote-control socket.

**Avoid when:**
- You target headless servers, minimal VMs, or GPU-less environments — a
  CPU-rendered emulator is safer.
- Copyleft (GPL-3.0) is a problem for embedding or redistribution.
- You need broad platform reach including Windows natively (kitty does not
  support Windows).
- You want a project governed by community consensus and generous feature
  intake rather than a single maintainer's judgment.

## Alternatives

- ghostty-org/ghostty — GPU terminal (Zig) that adopts kitty's graphics and
  keyboard protocols; use when you want kitty-class features with a permissive
  license and native platform UI.
- alacritty/alacritty — minimal GPU terminal (Rust); use when you want raw
  rendering speed and a small feature surface, and will run tmux for
  multiplexing.
- wez/wezterm — GPU terminal + multiplexer (Rust) with Lua config and native
  Windows support; use when you need cross-platform-including-Windows and a
  built-in mux.
- gnachman/iTerm2 — mature macOS-only emulator; use when you want deep macOS
  integration and a long feature list over cross-platform parity.
- foot — lightweight Wayland-native terminal (hosted on Codeberg); use on
  Wayland when you want low resource use without a GPU-heavy stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-10 | Repository created[^gh]. |
| 0.x | 2017 | First public releases; C+Python core, OpenGL glyph rendering. |
| graphics protocol | ~2018 | Inline image protocol introduced (approx.). |
| keyboard protocol | ~2021 | Comprehensive keyboard protocol with progressive enhancement (approx.). |
| Go kittens | early 2020s | Go added for the `ssh` kitten and later helpers (approx.). |

Precise per-release dates for protocol milestones are approximate; consult the
project changelog for exact versions[^changelog].

## References

[^gh]: GitHub API repository metadata, kovidgoyal/kitty — created 2016-10-16, license GPL-3.0. https://github.com/kovidgoyal/kitty
[^issues]: kitty issue-handling policy and maintainer triage practice. https://sw.kovidgoyal.net/kitty/support/
[^changelog]: kitty changelog / release history. https://sw.kovidgoyal.net/kitty/changelog/

## Tags

terminal-emulator, gpu-rendering, opengl, python, c, golang, cross-platform, cli, developer-tools, gplv3, graphics-protocol
