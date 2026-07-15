# bootandy/dust

> A `du` replacement in Rust that shows where disk space goes as a color-coded, terminal-height-aware tree instead of a wall of numbers.

[GitHub repo](https://github.com/bootandy/dust) ·
[License: Apache-2.0](https://github.com/bootandy/dust/blob/master/LICENSE)

## Overview

Dust ("du + rust") is a command-line disk-usage tool by Andy Boot, first committed in 2018[^1]. It answers one question — "what is eating my disk?" — and refuses to answer much else. Running `dust` in a directory prints a tree of the largest subdirectories and files, each with a percentage and an ASCII bar, sized to fit your terminal. There are no required flags: no `-d` for depth, no `-h` for human-readable, no piping through `sort` and `head`. That opinionated defaults-first design is the whole product.

The tool sits in the modern Rust CLI-replacement cohort alongside ripgrep, fd, bat, and eza — single-binary, fast, prettier output than the coreutils original. As of 2026 it has roughly 12,000 GitHub stars and is packaged nearly everywhere (Homebrew, cargo, snap, conda-forge, scoop, apt derivatives)[^2]. It is actively but calmly maintained; the last push at time of writing was February 2026, and the issue count stays low, which is typical for a tool whose scope is deliberately fixed.

The defining tension is **intelligence vs. predictability**. Dust does not show you every directory to a fixed depth; it heuristically picks the largest entries and recurses down only where the space actually is. That is exactly what you want when hunting for a runaway folder, and exactly what you don't want when you need stable, scriptable output — for which you must reach for `-j` (JSON) instead of parsing the visual tree.

## Getting Started

```bash
# crates.io — note the package is du-dust, the binary is dust
cargo install du-dust

# or Homebrew (macOS/Linux)
brew install dust

# or the install script
curl -sSfL https://raw.githubusercontent.com/bootandy/dust/refs/heads/master/install.sh | sh
```

```bash
dust                     # largest entries under the current directory
dust /var/log ~/Downloads  # multiple roots
dust -d 3                # cap displayed depth at 3 levels
dust -n 30               # show 30 entries instead of terminal-height default
dust -F                  # find your largest files (files only)
dust -e '\.log$'         # only include paths matching a regex
dust -j | jq             # machine-readable JSON for scripting
```

## Architecture / How It Works

Dust runs in two phases. First it walks the filesystem from each root, collecting a size for every node; the traversal is multi-threaded so many directories are `stat`-ed concurrently. Second, it folds those sizes into a tree, selects the largest entries that will fit the available terminal height, and renders each as a bar.

Several design choices are worth understanding because they change the numbers you see:

- **Disk usage, not file length, by default.** Dust reports actual blocks consumed (like `du`), not the apparent byte length of files. `-s` (`--apparent-size`) switches to file length. For sparse files, compressed filesystems, or many tiny files, the two differ significantly, and the default is the one that matches `du` rather than `ls -l`.
- **Hard links are counted once.** An inode reached through multiple hard links is charged a single time, so totals don't inflate. `-f` counts by inode (file count) and `-f -s` opts back into counting duplicates.
- **Heuristic recursion.** Rather than a fixed `-d` depth, dust descends preferentially into the branches holding the most space, so a deeply nested large file surfaces without you knowing where to look. This is the core differentiator over `ncdu`/`baobab`, which show directory sizes but hide *which file* inside a large directory is the culprit.
- **Bar shading encodes hierarchy.** The grey gradients on the left of each bar indicate parent membership: a subfolder shares a lighter grey line running up to its parent, so you can see at a glance that deleting one top-level folder reclaims everything beneath it.

Because collection is a recursive descent, extremely deep trees can exhaust the thread stack and produce `fatal runtime error: stack overflow`. Dust exposes `-S` to set a custom stack allocation as the escape hatch, which is an unusual flag to need and a tell that the walker is genuinely recursive rather than iterative.

## Production Notes

- **The default numbers are disk blocks.** If dust disagrees with `ls`, `find -size`, or a cloud provider's byte count, that is almost always disk-usage vs. apparent-size, not a bug. Pass `-s` to compare apples to apples.
- **Parallelism is a double edge.** Concurrent `stat` calls make dust fast on local SSDs, but on network filesystems (NFS, SMB, sshfs) or spinning disks the same concurrency can hammer the mount and run *slower* than plain `du`. Use `-x` to stay on one filesystem and avoid accidentally recursing into network or pseudo mounts.
- **Output is not a stable API.** The tree is a visual artifact tuned to terminal size and the `-n` heuristic; column widths, truncation, and which entries appear all change with the window. For anything programmatic, use `-j` and parse JSON — do not `grep`/`awk` the rendered tree.
- **`-n` is not `head -n`.** It sets how many of the largest entries to consider, and dust still chooses intelligently among them; it is not a simple top-N truncation. Increasing `-n` (e.g. `dust -n 90`) reveals more of the long tail.
- **Permissions are quiet by design.** Dust prints at most one "did not have permissions" message no matter how many directories it couldn't read, so a run over `/` as a normal user silently under-counts protected trees rather than flooding stderr.
- **Snap is sandboxed.** The snap package can only read files under `/home`; scanning `/var`, `/opt`, or `/` will appear empty. Use cargo, Homebrew, or a release binary for whole-system scans[^2].
- **Windows.** The GNU build works out of the box; the MSVC build needs the Visual C++ redistributable (`VCRUNTIME140.dll`).
- **Config file.** Persistent defaults live in `~/.config/dust/config.toml` (or `~/.dust.toml`), e.g. `reverse=true`, so team-shared or dotfile-managed defaults don't require wrapper aliases.

## When to Use / When Not

**Use when:**
- You need to find *where* space went quickly and interactively, and want the largest files surfaced without knowing their depth.
- You want a zero-config, single-binary `du` upgrade with readable, colored output.
- You are scripting disk reports and can consume its JSON (`-j`) output.

**Avoid when:**
- You need an interactive TUI to browse and delete directories — dust prints and exits; use `ncdu`, `dua interactive`, or `diskonaut`.
- You are scanning large network filesystems where parallel `stat` storms are undesirable — plain `du` is gentler.
- You need output whose exact format is contractually stable across versions and terminal sizes — parse `-j`, or stick with `du`.

## Alternatives

- Byron/dua-cli — Rust; parallel like dust but adds a full interactive TUI mode for browsing and deleting. Use when you want to act on results, not just view them.
- KSXGitHub/parallel-disk-usage — Rust (`pdu`); similar bar-chart tree, strong emphasis on parallelism and configurable output. Use when you want a close dust alternative with different display tuning.
- dundee/gdu — Go; interactive `ncdu`-style TUI optimized for SSD throughput. Use when you want fast interactive navigation.
- imsnif/diskonaut — Rust; treemap-style TUI you can navigate and delete from. Use when a spatial map beats a text tree.
- ncdu — C; the long-standing curses interactive standard. Use when you want the battle-tested TUI and don't need dust's largest-file surfacing.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-03-16 | First commit; "du + rust" concept[^1]. |
| — | ongoing | Packaged across Homebrew, cargo (`du-dust`), snap, conda-forge, scoop, deb-get. |
| current | 2026-02-21 | Latest push at time of writing; ~12k stars, low open-issue count[^2]. |

## References

[^1]: bootandy/dust README — "du + rust = dust. Like du but more intuitive." https://github.com/bootandy/dust
[^2]: GitHub repository metadata (stars, forks, license, last push) and packaging status via Repology (`du-dust`). https://github.com/bootandy/dust · https://repology.org/project/du-dust/versions

## Tags

rust, cli, disk-usage, du-replacement, filesystem, terminal, devtools, sysadmin, storage, command-line
