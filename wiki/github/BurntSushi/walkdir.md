# BurntSushi/walkdir

> Cross-platform Rust library for walking a directory tree recursively, with symlink following, file-descriptor budgeting, and cheap subtree pruning.

[GitHub repo](https://github.com/BurntSushi/walkdir) ·
[Docs](https://docs.rs/walkdir/) ·
[License: MIT OR Unlicense](https://github.com/BurntSushi/walkdir/blob/master/LICENSE-MIT)

## Overview

walkdir is a small, single-purpose crate by Andrew Gallant (BurntSushi, author of ripgrep) that turns a root path into an iterator over every entry beneath it. It exists because `std::fs::read_dir` only lists one directory level; walkdir handles the recursion, the ordering, the symlink-loop detection, and the syscall economy that a naive recursive `read_dir` gets wrong. It has been a load-bearing dependency of the Rust ecosystem for close to a decade and, at ~1,500 stars, is under-starred relative to how widely it is actually pulled in transitively[^1].

The defining design choice is that walkdir is an *iterator*, not a callback-driven walker like C's `nftw` or a channel-based streamer. Everything — depth limits, sorting, pruning, error handling — is expressed through the standard `Iterator` interface and a builder. This keeps the API tiny but means all traversal is single-threaded and lazy: you pay for each directory only as you pull entries from it. For parallel traversal you reach for a different crate (see Alternatives).

The crate is deliberately feature-frozen. It does not do glob matching, gitignore filtering, or content search; those live in sibling crates (`globset`, `ignore`) that the same author maintains. walkdir's job ends at "hand me the entries efficiently and correctly," which is why its issue tracker and release cadence are quiet — quiet here means finished, not abandoned.

## Getting Started

```toml
[dependencies]
walkdir = "2"
```

```rust
use walkdir::WalkDir;

// Recursively print every path under "foo", propagating errors.
for entry in WalkDir::new("foo") {
    let entry = entry.unwrap();
    println!("{}", entry.path().display());
}
```

```rust
use walkdir::{DirEntry, WalkDir};

// Prune hidden files/dirs *before* descending into them — cheap, no wasted syscalls.
fn is_hidden(e: &DirEntry) -> bool {
    e.file_name().to_str().map_or(false, |s| s.starts_with('.'))
}

let walker = WalkDir::new("foo").into_iter();
for entry in walker.filter_entry(|e| !is_hidden(e)).filter_map(Result::ok) {
    println!("{}", entry.path().display());
}
```

## Architecture / How It Works

Internally the walk is *not* stack-recursive — a deep tree will not blow the call stack. `WalkDir::into_iter()` returns an `IntoIter` that maintains an explicit stack of open `ReadDir` handles, one frame per directory level currently being traversed. Pulling the next item advances the deepest open handle; when it is exhausted, the frame is popped and its file descriptor closed. This is what lets `max_open` bound the number of simultaneously held directory handles: when the budget is hit, walkdir closes deeper handles and later re-opens and fast-forwards them, trading syscalls for descriptor pressure[^2].

`DirEntry` is the other performance-critical piece. On platforms where the directory read already returns the file type (Linux `d_type`, most BSDs), walkdir carries that type through without an extra `stat`. `entry.file_type()` is therefore usually free; `entry.metadata()` is the call that may hit the filesystem again. This distinction matters at scale — the difference between one syscall and two per entry across three million entries is the whole performance story.

Symlink handling is opt-in via `follow_links(true)`. With following enabled walkdir must guard against cycles, which it does using device/inode identity (the author's `same_file` crate) to detect when a followed link would re-enter an ancestor; such a case surfaces as an error carrying `loop_ancestor()` rather than an infinite loop. Ordering is controllable (`sort_by`, `sort_by_key`, `sort_by_file_name`, or unspecified OS order), as is traversal shape (`min_depth`, `max_depth`, `contents_first` for leaves-before-parents, `same_file_system` to stop at mount boundaries).

Errors are first-class iterator items: the element type is `Result<DirEntry, walkdir::Error>`, so a permission-denied directory yields an `Err` and the walk continues rather than aborting. `walkdir::Error` wraps an `io::Error` but adds context (`.path()`, `.depth()`, `.loop_ancestor()`) that the raw IO error lacks.

## Production Notes

- **Single-threaded by design.** walkdir will not saturate a multi-core box or a high-latency network filesystem. On spinning disks or NFS where wall-clock is dominated by IO wait, a parallel walker (`jwalk`, or `ignore`'s `WalkParallel`) can be several times faster. On a local SSD walkdir is already at the syscall floor and parallelism buys little.
- **`filter_entry` vs `filter`.** Use `filter_entry` to prune, not `Iterator::filter`. `filter_entry` prevents descent into a rejected *directory*; a plain `filter` still walks the whole subtree and discards entries afterward. On a tree with large ignored directories (`node_modules`, `.git`, `target`) this is the difference between a fast walk and a slow one.
- **Metadata cost is platform-dependent.** Code that calls `entry.metadata()` on every entry reintroduces a per-entry `stat` and can roughly double syscalls. Prefer `entry.file_type()` when you only need to distinguish files/dirs/symlinks. On Windows, `DirEntry` already carries metadata from the enumeration, so the tradeoff differs from Unix.
- **Symlink loops only error when following.** Without `follow_links(true)` a symlink is yielded as a symlink and never traversed, so loops are impossible; enabling following shifts loop detection cost (an inode identity check) onto every followed directory.
- **Ordering is nondeterministic by default.** The OS returns directory entries in arbitrary order. Tests or reproducible output must set an explicit `sort_by_*`; do not rely on incidental ordering that happens to appear on one filesystem.
- **MSRV moves in minor releases.** The minimum supported Rust is 1.60.0, and policy explicitly allows raising it in a minor (`2.y`) bump[^3]. Pin accordingly if you support old toolchains.

## When to Use / When Not

**Use when:**
- You need to recursively enumerate a filesystem tree and want correct symlink-loop handling and descriptor control for free.
- You want an `Iterator`-shaped API that composes with `filter_entry`, `filter_map`, and the rest of the iterator toolkit.
- You are walking a local disk where single-threaded traversal is already IO-bound.

**Avoid when:**
- You need `.gitignore`/`.ignore` semantics or glob filtering — use `ignore` or `globset` instead of reimplementing them on top of walkdir.
- You need parallel traversal to hide filesystem latency (network mounts, cold caches, huge trees) — reach for `jwalk` or `ignore`'s parallel walker.
- You only ever list a single directory — plain `std::fs::read_dir` is enough and drags in no dependency.

## Alternatives

- `BurntSushi/ripgrep` (the `ignore` crate) — same author; adds gitignore-aware filtering and a parallel walker. Use when you want ripgrep-style ignore semantics or multi-threaded traversal.
- `Byron/jwalk` — parallel recursive walk built on rayon. Use when traversal is latency-bound and you can spend cores to hide IO wait.
- `rust-lang/rust` (`std::fs::read_dir`) — the standard library primitive. Use when a single directory level is all you need.
- `rust-lang/glob` — pattern-based path matching rather than full-tree iteration. Use when you want `src/**/*.rs`-style selection, not exhaustive enumeration.
- `sharkdp/fd` — a CLI tool, not a library. Use when you want directory walking from the shell rather than embedded in Rust code.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-09-17 | Repository created; early 0.x releases[^1]. |
| 1.x | 2016 | First stable line with the recursive iterator. |
| 2.0 | 2018 | Builder redesign, `DirEntry`, `filter_entry`, richer error type. |
| 2.x | 2020–2026 | Maintenance line; MSRV policy formalized at 1.60.0[^3]. Last push 2026-06-23[^1]. |

## References

[^1]: GitHub API, `repos/BurntSushi/walkdir` — 1,519 stars, 130 forks, created 2015-09-17, last push 2026-06-23, license reported as Unlicense (crate is dual MIT OR Unlicense). Retrieved 2026-07-15. https://github.com/BurntSushi/walkdir
[^2]: walkdir API docs — `WalkDir::max_open`, `IntoIter`, and `DirEntry` behavior. https://docs.rs/walkdir/
[^3]: walkdir README, "Minimum Rust version policy" — MSRV 1.60.0, may increase in minor updates. https://github.com/BurntSushi/walkdir#minimum-rust-version-policy

## Tags

rust, filesystem, directory-traversal, recursion, iterator, cli-tooling, cross-platform, library, symlinks
