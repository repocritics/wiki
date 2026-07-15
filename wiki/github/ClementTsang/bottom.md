# ClementTsang/bottom

> A cross-platform terminal system monitor in Rust — time-series graphs for CPU, memory, network, and I/O next to a searchable process table.

[GitHub repo](https://github.com/ClementTsang/bottom) ·
[Official website](https://bottom.pages.dev) ·
[License: MIT](https://github.com/ClementTsang/bottom/blob/main/LICENSE)

## Overview

bottom (`btm`) is a graphical process and system monitor for the terminal, written in Rust and running on Linux, macOS, and Windows. It sits in the same niche as `top`/`htop` but leans on the "dashboard" style pioneered by gtop and gotop: the screen is a grid of widgets, several of which plot metrics over time as scrolling braille graphs rather than static number columns[^1]. The name and binary are a deliberate joke on `top`.

The project is the work of a single primary maintainer, Clement Tsang, active since 2019 and still shipping regular releases as of mid-2026[^2]. With ~13.7k stars it is one of the better-known entries in the Rust CLI-tooling wave (ripgrep, fd, bat, eza, dust) and is packaged almost everywhere — official repos for Arch, Alpine, Gentoo, Nixpkgs, Homebrew, Fedora COPR/Terra, Chocolatey, Scoop, winget, and conda-forge, plus prebuilt `.deb`/`.rpm`/`.msi` artifacts per release.

The defining tradeoff is scope versus footprint. bottom aims for one configurable tool that visualizes CPU, memory, temperatures, disk I/O, network, battery, and processes across three OSes. That breadth means a data-collection loop that samples many subsystems on a timer, so the monitor itself is not free — on busy machines or at aggressive refresh rates it uses noticeably more CPU than a plain `top`. If you want a graphing dashboard, that is the cost; if you want the lightest possible process list, it is overkill.

## Getting Started

```bash
# From crates.io (needs a recent stable Rust toolchain)
rustup update stable
cargo install bottom --locked

# Or via a system package manager
brew install bottom          # macOS / Linuxbrew
sudo pacman -S bottom        # Arch
nix profile install nixpkgs#bottom
winget install Clement.bottom
```

```bash
# Run it
btm

# htop-style compact "basic" mode, no graphs
btm --basic

# Group processes as a tree, refresh every 250ms
btm --tree --rate 250
```

Inside the UI: `?` shows keybindings, `/` searches the process widget, `dd` sends a kill signal to the selected process, `e` expands the focused widget to fill the screen, and `Tab`/arrow keys move focus. A config file is generated on first launch under the platform config directory (e.g. `~/.config/bottom/bottom.toml`) and controls layout, colors, and defaults.

## Architecture / How It Works

bottom is a single-binary TUI. Rendering is done with the `tui-rs`/`ratatui` widget family over a `crossterm` terminal backend, which is what gives it cross-platform terminal handling and the braille-based line graphs. Time-series widgets keep a ring of recent samples and redraw on each frame; the graph resolution is why a font with braille glyph coverage matters (missing glyphs render graphs as boxes).

Data collection runs on a background thread on a timer (the `--rate` refresh interval, default 1000 ms), decoupled from the draw loop so input stays responsive while samples are gathered. Cross-platform metrics come largely through the `sysinfo` crate plus platform-specific code paths, so exactly which sensors are available depends on the OS — Linux exposes the most (per-core CPU, temperatures, disk I/O, network), while macOS and Windows have gaps around temperatures and some I/O counters.

The screen is a **layout tree** of rows and columns of widgets defined in the config file. Each widget type (CPU, mem, net, temp, disk, battery, process) is independently sized and can be duplicated, reordered, filtered, or removed. Two rendering modes exist: the default dashboard, and `--basic`, an htop-like compact readout with no graphs.

The process widget is the most feature-dense: sortable columns, a search box (`/`) supporting regex, case-sensitivity, and whole-word toggles, filtering by name or command, tree grouping (`--tree`) that reparents children under their parent PID, and signal-sending for process termination. CPU accounting follows the "sum across cores" convention by default (a fully-busy 8-core process reads ~800%); a normalization option divides by core count to a 0–100% scale.

## Production Notes

- **The monitor's own overhead is real.** Sampling many subsystems every tick costs CPU; a low `--rate` (e.g. 250 ms) on a many-core box with thousands of processes makes `btm` itself show up in its own process list. Raise the refresh interval on constrained hosts or over SSH. This is inherent to the graphing approach, not a regression.
- **macOS metrics are partial.** Temperature sensors and some per-process or I/O figures are limited or unavailable, and Apple Silicon sensor coverage is weaker than Intel's. Don't expect Linux-level detail.
- **Battery is a compile-time feature.** The battery widget is gated behind a Cargo feature; official prebuilt binaries and most distro packages include it, but a hand-rolled `cargo install` with a trimmed feature set can omit it.
- **Braille font dependency.** Graphs are drawn with Unicode braille characters. Terminals/fonts without full braille coverage render the graphs as tofu boxes — a common "it looks broken" report that is actually a font issue.
- **Toolchain freshness for `cargo install`.** The README explicitly recommends `rustup update stable` first; bottom tracks recent stable Rust and older toolchains may fail to build. Use `--locked` to build against the tested dependency versions.
- **Snap needs interface connections.** The snap package must be granted `mount-observe`, `hardware-observe`, `system-observe`, and `process-control` or it can't read the metrics it's meant to show.
- **Config lives in TOML and can drift across versions.** New widget/config keys appear over releases; the generated default is the safest starting point after a major bump.

## When to Use / When Not

**Use when:**
- You want a terminal dashboard with historical CPU/mem/net/I/O graphs, not just an instantaneous table.
- You work across Linux, macOS, and Windows and want one tool with one config.
- You want a searchable, tree-capable process view with kill support and configurable layout.
- Single-binary distribution and wide package availability matter.

**Avoid when:**
- You need the absolute lightest-weight process viewer — `htop` or plain `top` cost less than a graphing monitor.
- You're on macOS/Windows and specifically need rich temperature or per-process I/O data that those platforms don't fully expose.
- You want remote/exportable metrics, alerting, or a web UI — bottom is a local interactive TUI, not a monitoring pipeline (use Glances or Prometheus-style tooling).

## Alternatives

- aristocratos/btop — C++ TUI monitor with a mouse-driven UI and denser theming; visually richer, heavier, no Rust-toolchain build.
- htop-dev/htop — the classic C process monitor; lighter and ubiquitous, but no over-time graphs and Unix-only.
- xxxserxxx/gotop — the Go predecessor bottom was inspired by; similar dashboard idea, far less actively maintained now.
- nicolargo/glances — Python monitor with web UI, REST/exporters, and remote mode; use it when you need monitoring integration, not just an interactive view.
- Use plain `top`/`procps` when you only need a quick, zero-dependency process list already present on the box.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0-alpha | 2019 | First public alpha; repo created Aug 2019[^2]. |
| 0.4.x | 2020 | Broader widget set and config maturity. |
| 0.5.0 | 2020-11-21 | Layout/config expansion. |
| 0.6.0 | 2021-05-09 | Feature and rendering work[^3]. |
| 0.7.0 | 2023-01-01 | Post-gap release cadence resumes. |
| 0.9.0 | 2023-05-10 | Continued widget/process improvements. |
| 0.10.0 | 2024-08-01 | Data-collection and platform updates. |
| 0.11.0 | 2025-08-06 | Feature release. |
| 0.12.0 | 2025-12-25 | Feature release. |
| 0.13.0 | 2026-06-20 | Feature release. |
| 0.14.0 | 2026-06-20 | Current line; latest patch 0.14.4 (2026-07-09)[^3]. |

## References

[^1]: bottom README — features, inspiration from gtop/gotop/htop, and supported platforms. https://github.com/ClementTsang/bottom
[^2]: bottom repository metadata (created 2019-08-28, MIT, Rust), retrieved via GitHub API 2026-07-15. https://github.com/ClementTsang/bottom
[^3]: bottom releases — dates for 0.5.0 through 0.14.4 taken from the GitHub Releases API, retrieved 2026-07-15. https://github.com/ClementTsang/bottom/releases

## Tags

rust, terminal, tui, system-monitor, process-monitor, cli, cross-platform, htop-alternative, observability, ratatui
