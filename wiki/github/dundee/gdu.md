# dundee/gdu

> Parallel disk usage analyzer for the terminal, written in Go — an ncdu
> replacement built for SSDs.

[GitHub repo](https://github.com/dundee/gdu) ·
[License: MIT](https://github.com/dundee/gdu/blob/master/LICENSE.md)

## Overview

Gdu ("go DiskUsage()") is a console disk usage analyzer by Daniel Milde,
written in Go. It answers the perennial "what is eating my disk" question with
an interactive TUI in the ncdu mold: scan a directory, browse the tree sorted
by size, drill in, delete offenders. Its differentiating bet is parallelism —
it walks the filesystem with many concurrent goroutines, which pays off on
SSDs and NVMe where random reads are cheap. The project's own benchmarks show
it roughly 7× faster than ncdu and `du -hs` on a 90G/500k-entry tree, and
within ~5% of diskus, the fastest tool measured[^1]. On rotating disks the
parallel walk can seek-thrash; a `--sequential` flag exists for that case.

The tool sits between two poles: simpler summarizers (diskus, dust) that only
print results, and heavier indexers (duc) that maintain a database. Gdu does
the interactive-browser job first, but has grown optional persistence (SQLite
or BadgerDB via `--db`), JSON export/import, archive browsing, and mtime
filtering — feature accretion that makes it closer to a general
disk-forensics tool than a one-shot analyzer.

Development is active and long-running: the repo dates to 2018, v1.0.0
shipped December 2020[^2], and releases continue steadily (v5.36.1 in April
2026, pushes as of July 2026). It is a single-maintainer project by commit
gravity, though it accepts outside contributions (~214 forks). At ~5.8k stars
it is well known but not the category default — ncdu retains that mindshare.

## Getting Started

Prebuilt binaries are on the releases page; most package managers carry it
(apt, dnf, pacman, apk, Homebrew, scoop, pkg)[^3]:

```bash
# Linux, from release tarball
curl -L https://github.com/dundee/gdu/releases/latest/download/gdu_linux_amd64.tgz | tar xz
chmod +x gdu_linux_amd64 && sudo mv gdu_linux_amd64 /usr/bin/gdu

# macOS — note: installed as gdu-go to avoid clashing with coreutils' GNU du
brew install gdu
```

```bash
gdu                     # interactive TUI on current directory
gdu -d                  # overview of all mounted disks
gdu --no-delete /data   # browse read-only; deletion disabled
gdu -npc / | head       # non-interactive stats, plain output, for scripts
gdu -o report.json /    # export full analysis as JSON
gdu -f report.json      # browse a previously exported analysis
```

In the TUI: arrows or `hjkl` navigate, `d` deletes the selection, `e` empties
a directory, `s`/`n` sort by size/name, `?` shows help.

## Architecture / How It Works

Gdu is a Go binary built on the tview/tcell TUI stack. The core is a
concurrent directory walker: each directory is scanned in its own goroutine
(bounded by `--max-cores`), and results aggregate up into an in-memory tree of
directory/file nodes. Hard links are deduplicated (counted once, flagged `H`);
scan errors degrade gracefully with per-entry flags (`!`, `.`) rather than
aborting[^3].

There are three distinct execution modes, and the analyzer actually swaps
implementations between them:

- **Interactive** (TTY detected): full tree in memory, browsable UI.
- **Non-interactive** (piped output or `-n`): without `--top`/`--depth`, gdu
  uses a memory-efficient analyzer that keeps only top-level totals, so
  memory stays constant regardless of tree size. Adding `--top` or `--depth`
  silently switches back to the full in-memory tree[^3].
- **Export** (`-o`): serializes the whole analysis to JSON for later
  offline browsing with `-f`.

Optional storage backends (`--db=file.sqlite` or `--db=file.badger`) write
analysis to disk instead of RAM, with `-r` to re-open a saved scan without
re-walking. This trades a lot of speed for persistence — the project's own
benchmarks put the BadgerDB-backed scan at ~6× slower cold and ~58× slower
warm than the in-memory scan[^1].

Deletion is normally synchronous and blocks the UI; opt-in config flags
`delete-in-background` and `delete-in-parallel` change that and are marked
experimental. `--enable-profiling` exposes pprof on localhost:6060.

## Production Notes

- **The `d` key deletes for real.** Gdu is a browser with a loaded gun; on
  servers, run `gdu --no-delete` (and `--no-view-file`, `--no-spawn-shell` if
  you are handing it to others) or set them in the config file. This is the
  single biggest operational footgun.
- **Memory scales with tree size in interactive mode.** The full node tree
  lives in RAM. On filesystems with tens of millions of entries, expect
  multi-GB usage; the constant-memory analyzer only engages in
  non-interactive mode without `--top`/`--depth`.
- **HDDs: use `--sequential`.** The default parallel walk is tuned for SSDs;
  on rotating disks it can be slower than a sequential scan.
- **Scanning `/`**: defaults ignore `/proc`, `/dev`, `/sys`, `/run`, but you
  usually also want `-x` (`--no-cross`) to stay on one filesystem — otherwise
  network mounts and bind mounts inflate results and slow the scan.
- **Apparent size vs disk usage.** Default is allocated blocks (like `du`);
  `-a` switches to apparent size. Sparse files and compressed filesystems
  (btrfs, ZFS) make the two diverge substantially.
- **TTY detection changes behavior.** Piped output flips gdu into
  non-interactive mode with different output shape; `--interactive` forces
  the TUI even when piped.
- **The `--db` path is not a speedup.** Persistent storage exists for
  save/reload workflows, not performance; benchmarks show order-of-magnitude
  slowdowns versus in-memory scans[^1].

## When to Use / When Not

**Use when:**
- You want an interactive, keyboard-driven "find and delete the big stuff"
  workflow on servers or workstations with SSDs.
- You need to scan on one machine and browse elsewhere (JSON export/import).
- You want a single static binary that installs anywhere Go targets.

**Avoid when:**
- You only need a number in a script — `diskus` or plain `du -sh` is simpler
  and (for diskus) marginally faster.
- You are on memory-constrained hosts scanning huge trees interactively —
  ncdu's C implementation has a lower memory footprint per node.
- You need continuously updated usage data over time — an indexer like duc
  or filesystem-native accounting (btrfs qgroups, ZFS) fits better.

## Alternatives

- ncdu — the C/zig incumbent with the same UI concept; smaller memory
  footprint and ubiquity in distro repos, but single-threaded scanning.
- sharkdp/diskus — use instead when you only want the total, as fast as
  possible, with no UI.
- Byron/dua-cli — Rust, gdu-like interactive mode; use if you prefer the
  Rust toolchain or its aggregate mode.
- bootandy/dust — use for a one-shot visual tree summary in the terminal
  rather than interactive browsing.
- KSXGitHub/parallel-disk-usage — Rust parallel scanner with tree output;
  closest to gdu's performance approach, no delete workflow.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2020-12-24 | First tagged release (repo existed since 2018)[^2]. |
| v2.0.0–v4.0.0 | 2021-01 | Rapid early iteration; three major bumps in one month. |
| v5.0.0 | 2021-05 | Major-version line still current five years later. |
| v5.30.1 | 2024-12 | Steady maintenance cadence through 2024. |
| v5.35.0 | 2026-03 | Continued feature releases (mtime filters, archive browsing era). |
| v5.36.1 | 2026-04 | Latest release as of this writing[^2]. |

## References

[^1]: gdu README, benchmark tables (hyperfine, 90G / 100k dirs / 400k files, cold and warm cache). https://github.com/dundee/gdu#benchmarks
[^2]: gdu releases page. https://github.com/dundee/gdu/releases
[^3]: gdu README and INSTALL.md. https://github.com/dundee/gdu#installation

## Tags

go, cli, tui, disk-usage, filesystem, sysadmin, performance, ncdu-alternative, terminal
