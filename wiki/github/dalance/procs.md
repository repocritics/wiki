# dalance/procs

> A modern replacement for `ps` written in Rust — colored, human-readable process listing with search, tree view, and a `top`-like watch mode.

[GitHub repo](https://github.com/dalance/procs) ·
[License: MIT](https://github.com/dalance/procs/blob/master/LICENSE-MIT)

## Overview

procs is a command-line process viewer built to replace the venerable `ps`. Where `ps` exposes process state through a terse, cryptic flag grammar (BSD vs. UNIX vs. GNU styles) and monochrome fixed-width columns, procs ships colored output, multi-column keyword search, and information `ps` cannot surface at all: bound TCP/UDP ports, per-process read/write throughput, the owning Docker container name, and richer memory accounting[^1]. It sits in the same "rewrite a classic Unix tool in Rust" lineage as ripgrep, fd, bat, and eza — the goal is not new capability so much as a saner default experience.

The project began in early 2019[^2] and has stayed a single-maintainer effort by dalance, the author of a broader family of Rust tooling. It is deliberately scoped: procs is a viewer, not a process manager. It does not send signals, kill processes, or renice them. If you want an interactive killer or a full TUI dashboard, procs is not that — htop and btop occupy that niche, and procs' watch mode is a lighter, read-only alternative.

The defining tradeoff is platform depth versus breadth. procs is a native, per-OS reader of process tables, so the set of available columns differs sharply between Linux, macOS, Windows, and FreeBSD. Linux is the reference target with the fullest column set; macOS and FreeBSD are marked experimental, and many columns simply do not exist off Linux[^1]. Treat the column-support matrix in the README as the contract, not the feature list.

## Getting Started

```bash
# macOS / Linux via Homebrew
brew install procs

# Any platform with a Rust toolchain
cargo install procs
```

Distro packages exist for Arch (`pacman -S procs`), Fedora (`dnf install procs`), Alpine, Nix, Snap, Scoop, winget, and MacPorts[^1]. Prebuilt binaries and an RPM are attached to each GitHub release.

```console
# All processes, colored and paged automatically
procs

# Search: non-numeric matches USER/Command, numeric matches PID
procs zsh
procs --or 6000 60001 16723

# top-like live view, 2s refresh
procs --watch-interval 2

# Process dependency tree
procs --tree

# Sort descending by CPU
procs --sortd cpu
```

## Architecture / How It Works

procs is a Rust binary with no daemon and no persistent state. Each invocation snapshots the OS process table and renders it. The read path is entirely OS-specific: on Linux it parses `/proc`, and its ability to expose extras like read/write throughput and per-process signal masks comes directly from `/proc/<pid>/io`, `/status`, and friends. This is why the column matrix is so uneven — a column exists only where the underlying OS exposes the datum. macOS goes through the kernel's sysctl/libproc surface, Windows through its native process APIs, and FreeBSD through its own kvm/sysctl path.

Rendering is driven by a TOML configuration model. Columns are not hardcoded: the `[[columns]]` array in the config file is an ordered list of `kind` (Pid, Username, VmRss, TcpPort, Docker, State, ...) plus per-column style, alignment, width bounds, and search-eligibility flags[^1]. The left-to-right screen order is the array order. Built-in presets (`--use-config`) and a `--gen-config` dump let you start from a working baseline; `config/large.toml` in the repo is the historical default and shows the full palette-driven styling (`by_percentage`, `by_state`, `by_unit` color ramps).

Search is a first-class concept rather than a `grep` afterthought. Keywords are classified as numeric (exact match, PID by default) or non-numeric (partial match, USER/Command by default), and any column can be opted into either search class via `numeric_search` / `nonnumeric_search`. Multiple keywords combine with `--and`, `--or`, `--nand`, `--nor`, with the default settable in the `[search]` config section. The `--insert`, `Slot`, and `MultiSlot` mechanism lets you splice ad-hoc columns into a fixed layout at runtime without rewriting the config.

Watch mode is a minimal in-terminal loop (`n`/`p` to move the sort column, `a`/`d` for order, `q` to quit) rather than a full-screen TUI framework — closer to `watch procs` than to htop.

## Production Notes

- **Permissions gate the interesting columns.** On macOS a normal user cannot see other users' process info at all; on Linux, extras like read/write throughput are restricted per user[^1]. Ports only resolve for processes the current user owns. To get a complete picture you run under `sudo`, and the README documents a `NOPASSWD` sudoers entry for the procs binary if you want that without a password prompt — a convenience that is also a small privilege-escalation surface worth reviewing before deploying broadly.
- **Docker column needs socket access.** The `Docker` column appears only when procs can reach `unix:///var/run/docker.sock`. Docker Toolbox on macOS (no UNIX socket) is unsupported; Docker Desktop for Mac is supported but the maintainer notes it is untested[^1]. Granting the socket to procs effectively grants Docker daemon access — scope accordingly.
- **Platform maturity is uneven.** Linux is the tested reference. macOS is "experimentally supported" and validated only in CI, not on real hardware, so the maintainer explicitly solicits real-machine bug reports. FreeBSD is likewise experimental. Do not assume a column that works on Linux exists elsewhere — consult the per-OS matrix.
- **Config path moved and is OS-specific.** Files live under `~/.config/procs/` (Linux), `~/Library/Preferences/com.github.dalance.procs/` (macOS), or `~/AppData/Roaming/dalance/procs/` (Windows), with `/etc/procs/procs.toml` as a system fallback. A legacy `~/.procs.toml`, if present, overrides all of them — a subtle footgun when a stale dotfile silently wins over your intended config.
- **It is a viewer only.** There is no kill/signal/renice. Pair it with `kill`, `pkill`, or a separate tool; do not reach for procs expecting process control.
- **Still pre-1.0.** The version line has been in the 0.x range for years (0.14.x as of mid-2026)[^3]. In practice it is stable and widely packaged, but semver-wise the author has not committed to a 1.0 API/config-format freeze, so watch the CHANGELOG across minor bumps.

## When to Use / When Not

**Use when:**
- You want a readable, colored `ps` for interactive terminal use on a dev machine or server.
- You need port bindings, I/O throughput, or Docker container names correlated to processes in one view.
- You want quick keyword/tree/watch views without learning `ps` flag dialects.

**Avoid when:**
- You need to act on processes (kill, renice) — procs is read-only; use htop/btop or plain `kill`.
- You are parsing output programmatically in scripts — `ps -o` with stable field selection is more predictable than a display-oriented tool.
- You are off Linux and depend on a specific column — verify it exists on your OS first; macOS/FreeBSD support is experimental.

## Alternatives

- htop/htop — interactive TUI process viewer with kill/renice/tree; use when you need to act on processes, not just view them.
- aristocratos/btop — heavier, prettier full-screen resource monitor; use when you want CPU/mem/net/disk dashboards alongside processes.
- dalance/procs itself supersedes plain `ps`; keep `ps -o` when you need scriptable, stable machine-readable output.
- ClementTsang/bottom — cross-platform `top` alternative in Rust with graphs; use when you want charts and a TUI over a static listing.
- nix-community/pueue or plain `pgrep`/`pkill` — use when the actual need is querying or managing by name rather than a formatted table.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2019-01 | Project created; Linux-first `ps` replacement[^2]. |
| 0.9.x | ~2020 | Config model matured; `config/large.toml` was the pre-0.9.21 default[^1]. |
| 0.14.12 | 2026 | Current release line; broad distro packaging (Homebrew, Arch, Fedora, Nix, Snap, winget)[^3]. |

## References

[^1]: procs README — features, platform matrix, installation, configuration, and permission notes. https://github.com/dalance/procs/blob/master/README.md
[^2]: GitHub repository metadata, dalance/procs — created 2019-01-28, MIT license, ~6.1k stars as of 2026-07. https://github.com/dalance/procs
[^3]: procs CHANGELOG and crates.io release history (0.14.x line, mid-2026). https://github.com/dalance/procs/blob/master/CHANGELOG.md

## Tags

rust, cli, process-viewer, ps-replacement, system-monitoring, terminal, devtools, cross-platform, sysadmin, htop-alternative
