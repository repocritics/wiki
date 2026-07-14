# alacritty/alacritty

> A GPU-accelerated terminal emulator that does one window well and delegates everything else.

[GitHub repo](https://github.com/alacritty/alacritty) ·
[Official website](https://alacritty.org) ·
[License: Apache-2.0](https://github.com/alacritty/alacritty/blob/master/LICENSE-APACHE)

## Overview

Alacritty is a cross-platform terminal emulator written in Rust that renders its
grid with the GPU via OpenGL[^1]. It was started by Joe Wilm in 2016 and
announced publicly in January 2017[^2], with the explicit goal of being the
fastest terminal emulator available while staying simple. It runs on Linux, BSD,
macOS, and Windows (the latter via ConPTY on Windows 10 1809+).

The defining design decision is subtractive: Alacritty deliberately omits tabs,
window splits, a GUI config editor, and font ligatures. The maintainers'
position is that multiplexing belongs to a window manager or a terminal
multiplexer like tmux, so the terminal itself stays a single scrollback grid
plus a renderer[^3]. This is the project's central tension — it is fast and
predictable precisely because it refuses the feature surface that kitty and
WezTerm embrace, and whether that trade is acceptable depends entirely on
whether you already live in tmux or a tiling WM.

Despite ~65k stars and heavy daily use, the project still describes itself as
**beta**: a stable, widely-deployed beta, but one that has never cut a 1.0 and
carries a short list of intentional non-features. It is configured entirely
through a TOML file with no interactive setup.

## Getting Started

```bash
# macOS
brew install --cask alacritty
# Arch
pacman -S alacritty
# cargo (from source; needs a Rust toolchain + system deps)
cargo install alacritty
```

Alacritty does not write a config for you. Create one at
`$XDG_CONFIG_HOME/alacritty/alacritty.toml` (or `~/.config/alacritty/alacritty.toml`):

```toml
[window]
opacity = 0.95
padding = { x = 8, y = 8 }

[font]
size = 13.0
normal = { family = "JetBrains Mono" }

[colors.primary]
background = "#1e1e2e"
foreground = "#cdd6f4"

[[keyboard.bindings]]
key = "N"
mods = "Command"
action = "SpawnNewInstance"
```

Live-reload is on by default: saving the file re-applies most settings without a
restart.

## Architecture / How It Works

Alacritty is three cooperating layers:

1. **PTY + parser.** A pseudo-terminal feeds a byte stream into a VTE state
   machine (the `vte` crate, maintained in the same org) that decodes ANSI/VT
   escape sequences into grid mutations. The parsed model is a 2D grid of cells,
   each holding a character, colors, and flags.
2. **Terminal state.** The grid, scrollback ring buffer, selection, and vi-mode
   cursor live in `alacritty_terminal`, a library crate that is deliberately
   separable from rendering (other projects embed it).
3. **Renderer + windowing.** The frontend uses `winit` for cross-platform
   windowing/input and draws the visible grid through OpenGL (ES 2.0 minimum).
   Glyphs are rasterized once by `crossfont` (CoreText on macOS, FreeType +
   fontconfig on Linux, DirectWrite on Windows) into a GPU glyph atlas texture;
   each frame then draws cells as instanced textured quads. Redraw is
   damage-driven — the screen is only re-rendered when content or the cursor
   changes.

The speed story is mostly this: batched instanced rendering plus a tight parser,
not a magic algorithm. Because rendering is decoupled from the terminal model,
Alacritty avoids re-shaping text every frame — which is also why complex text
shaping features (ligatures, some grapheme clustering) are absent, since it does
not run a full HarfBuzz shaping pass per line.

Multi-window is handled in a single process via an event loop; `alacritty msg
create-window` uses an IPC socket to spawn additional windows in the running
instance rather than forking new processes.

## Production Notes

**No ligatures, by design.** This is the single most common surprise. Fonts like
Fira Code render, but their programming ligatures do not compose. There is no
config flag; it is an architectural choice. If you need ligatures, this is a hard
stop — use kitty, WezTerm, or Ghostty.

**Config format migrated YAML → TOML.** Alacritty 0.13 (early 2024) introduced
TOML configuration and deprecated YAML; 0.14 removed YAML support entirely[^4].
Old `alacritty.yml` files silently stop being read after upgrading. A migration
subcommand (`alacritty migrate`) converts existing configs. This bites users who
upgrade via a rolling package manager without reading release notes.

**OpenGL dependency.** Rendering requires a working OpenGL ES 2.0 context. On
headless servers, over some VNC/remote setups, in certain VMs, or under old/soft
GL drivers, Alacritty may fail to start or fall back badly. This is the usual
cause of "works on my laptop, not on the remote box" reports.

**Multiplexing is your problem.** No tabs and no splits means that for anything
beyond one shell you run tmux or a tiling window manager. Teams standardizing on
Alacritty effectively standardize on tmux alongside it; budget for that config
surface, keybinding overlap, and the scrollback interaction between the two.

**Scrollback and search.** Scrollback exists (added in 0.2, 2018) and there is a
built-in vi mode for keyboard selection and regex search, but the ergonomics are
spartan compared to kitty's scrollback pager or a multiplexer's copy mode.

**Performance is real but contextual.** Alacritty benchmarks its own throughput
with `vtebench`[^5] and generally leads on raw parse/draw throughput. In daily
use the difference against other GPU terminals is often imperceptible; the honest
framing is "never the bottleneck," not "visibly faster than everything."

## When to Use / When Not

**Use when:**
- You already live in tmux or a tiling WM and want the terminal to just be a fast grid.
- You want the same terminal and config across Linux, BSD, macOS, and Windows.
- You value a small, predictable, text-file-configured tool with no telemetry or GUI.
- You care about input-to-glyph latency and high-throughput output (log tailing, builds).

**Avoid when:**
- You need font ligatures, tabs, or splits inside the terminal itself.
- You want images/graphics protocols, a built-in pager, or session management out of the box.
- You run in environments without reliable OpenGL (some VMs, remote/soft-GL setups).
- You prefer a batteries-included terminal and don't want to also configure tmux.

## Alternatives

- kovidgoyal/kitty — GPU terminal with tabs, splits, ligatures, and a graphics protocol; use it when you want those features in the terminal itself.
- wez/wezterm — GPU terminal with built-in multiplexing and Lua config; use it when you want tmux-like features without tmux.
- ghostty-org/ghostty — GPU terminal (Zig) focused on native platform integration and low latency; use it when you want a feature-fuller modern alternative.
- kovidgoyal/... foot (Wayland) — lightweight CPU/GPU terminal; use it on Wayland when you want minimal resource use.
- tmux/tmux — not a terminal but the multiplexer Alacritty expects you to pair with; use it for tabs/splits/sessions on top of Alacritty.

## History

| Version | Date | Notes |
|---------|------|-------|
| public announce | 2017-01-06 | "Announcing Alacritty, a GPU-Accelerated Terminal Emulator"[^2]. |
| 0.2 | 2018-09 | Scrollback support added; benchmarks published[^6]. |
| 0.5 | 2020 | Vi mode, regex search, IPC (`alacritty msg`). |
| 0.13 | 2024 | TOML configuration introduced; YAML deprecated[^4]. |
| 0.14 | 2024 | YAML config support removed; `alacritty migrate` for conversion. |
| 0.15 | 2025 | Continued 0.x maintenance line (still beta, no 1.0). |

## References

[^1]: Repository description and topics, `alacritty/alacritty` (Rust, OpenGL, cross-platform), retrieved 2026-07-14. https://github.com/alacritty/alacritty
[^2]: Joe Wilm, "Announcing Alacritty, a GPU-Accelerated Terminal Emulator" — 2017-01-06. https://jwilm.io/blog/announcing-alacritty/
[^3]: Alacritty README, FAQ ("Why isn't feature X implemented?") — tabs/splits deferred to WM or tmux. https://github.com/alacritty/alacritty#faq
[^4]: Alacritty configuration docs / release notes on the TOML migration. https://alacritty.org/config-alacritty.html
[^5]: vtebench — Alacritty's terminal throughput benchmark tool. https://github.com/alacritty/vtebench
[^6]: Joe Wilm, "Alacritty Lands Scrollback, Publishes Benchmarks" — 2018-09-17. https://jwilm.io/blog/alacritty-lands-scrollback/

## Tags

terminal-emulator, rust, opengl, gpu-accelerated, cross-platform, cli, tmux, vte, developer-tools, macos, linux, windows
