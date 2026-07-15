# notify-rs/notify

> Cross-platform filesystem notification library for Rust — one API over inotify, FSEvents, kqueue, ReadDirectoryChangesW, and polling.

[GitHub repo](https://github.com/notify-rs/notify) ·
[Documentation](https://docs.rs/notify) ·
[License: CC0-1.0](https://github.com/notify-rs/notify/blob/main/notify/LICENSE-CC0)

## Overview

`notify` is the de facto filesystem-watching crate in the Rust ecosystem. It wraps each operating system's native change-notification API behind a single `Watcher` trait, and falls back to a polling implementation where no native API exists. It was born out of `cargo watch` and inspired by Go's fsnotify and Node's Chokidar[^1]. As of 2026 it has over 125 million crate downloads[^2] and sits underneath a large share of the Rust tooling people use daily: rust-analyzer, deno, zed, alacritty, mdBook, watchexec, cargo-watch, and — via Rust bindings — the Python `watchfiles` library[^1].

The defining tension of the project is that filesystem event APIs are genuinely different across platforms, and `notify` chooses a thin-abstraction philosophy: it normalizes *what an event looks like* (an `Event` with an `EventKind` and affected paths) but does not fully normalize *event semantics*. FSEvents coalesces and reports at directory granularity, inotify emits per-file events but does not recurse, and Windows' `ReadDirectoryChangesW` has its own buffer-overflow failure mode. The library exposes a common shape over these, but the caller still inherits platform-specific behavior at the edges — which is why most non-trivial consumers add a debouncer on top.

The 5.0 release (2022) was a ground-up API rewrite that dropped the old event model and, notably, removed built-in debouncing from the core crate[^3]. Debouncing now lives in two sibling crates, `notify-debouncer-mini` and `notify-debouncer-full`; the latter also does rename correlation and event coalescing using stable file identifiers from the `file-id` crate.

## Getting Started

```toml
# Cargo.toml
[dependencies]
notify = "8"
```

```rust
use notify::{RecommendedWatcher, RecursiveMode, Watcher, Event};
use std::sync::mpsc;
use std::path::Path;

fn main() -> notify::Result<()> {
    let (tx, rx) = mpsc::channel::<notify::Result<Event>>();

    // RecommendedWatcher selects the best backend for the target OS.
    let mut watcher = RecommendedWatcher::new(tx, notify::Config::default())?;
    watcher.watch(Path::new("."), RecursiveMode::Recursive)?;

    for res in rx {
        match res {
            Ok(event) => println!("{:?}: {:?}", event.kind, event.paths),
            Err(e) => eprintln!("watch error: {e:?}"),
        }
    }
    Ok(())
}
```

For most applications you want the debouncer instead of the raw stream:

```rust
use notify_debouncer_full::{new_debouncer, DebounceEventResult};
use std::time::Duration;

let (tx, rx) = std::sync::mpsc::channel();
let mut debouncer = new_debouncer(Duration::from_millis(500), None, tx)?;
debouncer.watch(".", notify::RecursiveMode::Recursive)?;
```

## Architecture / How It Works

`notify` is a Cargo workspace of small crates: `notify` (the watchers), `notify-types` (shared `Event`/`EventKind` definitions, split out so downstream crates avoid a hard dep on the full watcher), `notify-debouncer-mini`, `notify-debouncer-full`, and `file-id`.

The core abstraction is the `Watcher` trait. Concrete implementations map to a native backend:

- **Linux / Android** — `INotifyWatcher` over inotify. inotify has no native recursive mode, so `notify` walks the directory tree and registers one watch descriptor **per directory**. This is the origin of most Linux-side scaling problems.
- **macOS** — `FsEventWatcher` over FSEvents by default, or a kqueue-based watcher behind the `macos_kqueue` feature. FSEvents reports events at directory granularity with a latency window and coalesces rapid changes.
- **Windows** — `ReadDirectoryChangesW` with an internal buffer; overflow drops events.
- **BSD / iOS** — `KqueueWatcher`. kqueue requires an open file descriptor per watched file, so recursive watches are descriptor-hungry.
- **Everywhere** — `PollWatcher`, which stat-walks the tree on an interval and diffs metadata (optionally content hashes). Correct anywhere, cheap nowhere.

`RecommendedWatcher` is a type alias resolved at compile time to the best backend for the target. Events arrive as `Event { kind: EventKind, paths: Vec<PathBuf>, attrs }`, where `EventKind` is one of `Create`, `Modify`, `Remove`, `Access`, `Any`, or `Other`, each with more specific sub-variants. Crucially, the granularity and reliability of those sub-variants is backend-dependent — you should treat anything more specific than the top-level kind as a hint, not a contract.

`notify-debouncer-full` sits above a raw watcher and does the work `notify` deliberately does not: it buffers events over a time window, coalesces duplicates, and correlates rename-from / rename-to pairs by asking `file-id` for a stable inode/file-index identifier so a move looks like one logical operation instead of two unrelated events.

## Production Notes

**inotify watch exhaustion is the classic outage.** Recursive watches consume one `fs.inotify.max_user_watches` entry per directory; on large trees (`node_modules`, monorepos) you exhaust the default limit and `watch()` starts returning errors mid-tree. The fix is operational — raise `max_user_watches` via sysctl — not something the library can paper over. Any tool shipping `notify` on Linux should surface this error clearly rather than silently under-watching.

**Editors defeat naive watchers.** Atomic-save editors write to a temp file and rename it over the target. Depending on platform you'll see remove+create, a rename pair, or a bare modify — for the "same" logical save. This is the single biggest reason to use `notify-debouncer-full`: without rename correlation, downstream logic double-fires or loses track of the file identity.

**Newly created subdirectories race.** On inotify and kqueue, a directory created after the initial tree-walk must be discovered and watched; files created inside it in the same instant can be missed. Debouncer-full re-scans to narrow this window but cannot fully close it.

**FSEvents coarseness and latency.** On macOS the default backend reports which directories changed, not always which files, and batches with a latency delay. Code that needs precise per-file, low-latency events on macOS sometimes opts into the `macos_kqueue` feature, trading descriptor cost and different semantics for tighter granularity.

**Polling is a correctness fallback, not a scaling one.** `PollWatcher` re-stats the whole tree each interval. It is the right choice on network filesystems (NFS/SMB) and containers where native events are unreliable, but CPU and latency grow with tree size; content-hash mode multiplies that cost.

**Debouncing lives outside core (since v5).** Teams upgrading from v4 are frequently surprised that `notify` no longer debounces; you must add a debouncer crate and pick a window. Too short and you re-fire on partial writes; too long and you add perceptible lag to a watch-rebuild loop.

## When to Use / When Not

**Use when:**
- You need filesystem change events in Rust and want native backends with a single API.
- You're building watch-and-rebuild tooling (dev servers, live reload, linters, indexers).
- You want a fallback (polling) for environments where native events don't work.

**Avoid / reconsider when:**
- You need identical event *semantics* on every OS without writing platform-aware code — you'll still be smoothing over differences with a debouncer.
- You're watching very large trees on Linux without control over `max_user_watches`.
- You only care about one platform and its native crate (`inotify`, `kqueue`, `windows` APIs) directly would be simpler and give you full-fidelity events.
- You want desktop/toast notifications — that's `notify-rust`, an unrelated crate with a confusingly similar name.

## Alternatives

- inotify (hannobraun/inotify) — use directly when you target only Linux and want raw, full-fidelity inotify events without an abstraction layer.
- watchexec/watchexec — use when you want a ready-made CLI/library that already builds a debounced, filtered watch loop on top of notify.
- fsnotify/fsnotify — use when your project is in Go rather than Rust; it occupies the same niche and inspired notify.
- paulmillr/chokidar — use when you're in Node.js; the JavaScript counterpart notify was partly modeled on.
- watchdog (gorakhargosh/watchdog) — use for a mature cross-platform watcher in Python (though the Rust-backed watchfiles, built on notify, is the faster modern option).

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.0 | 2015-06-08 | Early cross-platform release. |
| 3.0 | 2016-10-30 | Backend and API iteration. |
| 4.0 | 2017-02-07 | Long-lived stable line; old event model with built-in debouncing[^2]. |
| 5.0 | 2022-08-30 | Ground-up API rewrite; new `Event`/`EventKind` model; debouncing removed from core into sibling crates[^3]. |
| 6.0 | 2023-05-17 | Config API and backend refinements. |
| 7.0 | 2024-10-25 | Continued breaking cleanup; `notify-types` split. |
| 8.0 | 2025-01-10 | Current major line (8.2.0, 2025-08-03). |
| 9.0-rc | 2026-01→ | Release-candidate line; MSRV raised to 1.88[^1]. |

## References

[^1]: notify README and MSRV policy, notify-rs/notify (main branch), retrieved 2026-07. https://github.com/notify-rs/notify
[^2]: crates.io crate metadata for `notify` (downloads, version release dates), retrieved 2026-07. https://crates.io/crates/notify
[^3]: notify API documentation and debouncer crates on docs.rs. https://docs.rs/notify and https://docs.rs/notify-debouncer-full

## Tags

rust, filesystem, file-watching, inotify, fsevents, kqueue, cross-platform, event-notification, developer-tools, library
