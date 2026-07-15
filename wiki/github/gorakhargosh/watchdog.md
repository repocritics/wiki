# gorakhargosh/watchdog

> Cross-platform Python library for watching filesystem events, with a uniform API over inotify, FSEvents, kqueue, and Windows change notifications.

[GitHub repo](https://github.com/gorakhargosh/watchdog) ·
[Documentation](https://python-watchdog.readthedocs.io/) ·
[License: Apache-2.0](https://github.com/gorakhargosh/watchdog/blob/master/LICENSE)

## Overview

Watchdog is a Python library that reports filesystem changes — file created, modified, moved, deleted — through a single event-handler API, regardless of the operating system underneath. It was started in 2010 by Yesudeep Mangalapilly (then at Google) and has been maintained since roughly 2018 by Mickaël Schoentgen[^1]. It is one of the most widely depended-on utility packages in the Python ecosystem: dev servers that reload on save (Flask's reloader, `uvicorn --reload`'s older backend, Django's `runserver` autoreload path), documentation tools like MkDocs and Sphinx-autobuild, and countless build/deploy scripts sit on top of it.

The library's central promise — and its central tension — is the abstraction itself. Each platform exposes a fundamentally different mechanism with different semantics: Linux `inotify` is per-directory and edge-triggered, macOS `FSEvents` coalesces and delays events at directory granularity, BSD/macOS `kqueue` needs one file descriptor per watched file, and Windows `ReadDirectoryChangesW` is a buffered async API. Watchdog papers over these into one `Observer` that emits `FileSystemEvent` objects. That uniformity is what makes it useful, and the leaks in the abstraction — missing modify events, duplicate events, coalesced renames, watch-limit exhaustion — are what fill its issue tracker.

Watchdog ships two things: the library (`watchdog`) and an optional CLI (`watchmedo`, installed via the `watchdog[watchmedo]` extra) that runs shell commands or YAML-defined "tricks" in response to events without writing any Python.

## Getting Started

```bash
python -m pip install -U watchdog
# CLI utility (pulls in PyYAML):
python -m pip install -U 'watchdog[watchmedo]'
```

```python
import time
from watchdog.events import FileSystemEvent, FileSystemEventHandler
from watchdog.observers import Observer


class MyEventHandler(FileSystemEventHandler):
    def on_any_event(self, event: FileSystemEvent) -> None:
        print(event)  # e.g. <FileModifiedEvent: src_path='./foo.py'>


observer = Observer()
observer.schedule(MyEventHandler(), ".", recursive=True)
observer.start()
try:
    while True:
        time.sleep(1)
finally:
    observer.stop()
    observer.join()
```

```bash
# watchmedo: run a command on every change to .py/.txt files
watchmedo shell-command \
    --patterns='**/*.py;**/*.txt' \
    --recursive \
    --command='echo "${watch_src_path}"' \
    .
```

## Architecture / How It Works

`Observer` is a facade. At import time watchdog selects a platform-specific implementation — `InotifyObserver`, `FSEventsObserver`, `KqueueObserver`, `WindowsApiObserver`, or the OS-independent `PollingObserver` — and exposes it under the common name `Observer`. Each observer runs a background thread that pulls native events, normalizes them into `FileSystemEvent` subclasses (`FileCreatedEvent`, `FileModifiedEvent`, `FileMovedEvent`, `DirCreatedEvent`, etc.), and dispatches them to the handlers registered via `schedule()`. Handlers run on the observer's thread, so blocking work in a callback stalls event delivery.

The per-platform reality behind the facade:

- **Linux (`inotify`)** — inotify watches a single directory, not a tree. For `recursive=True`, watchdog walks the tree and adds a watch descriptor for every subdirectory, and adds/removes watches as directories appear and disappear. Deep trees consume many watch descriptors, bounded by `fs.inotify.max_user_watches`.
- **macOS (`FSEvents`)** — the default. FSEvents is efficient and recursive natively, but historically reports at directory granularity with latency/coalescing; watchdog compensates by snapshotting and diffing directory contents to synthesize per-file events, which is where some duplicate/misattributed events originate.
- **BSD & macOS (`kqueue`)** — used on FreeBSD, available on macOS. kqueue monitors via file descriptors, one per watched file, so watching a large tree can exhaust the process FD limit (`ulimit -n`). The README itself calls this "not a very scalable way" to watch deep trees.
- **Windows (`ReadDirectoryChangesW`)** — uses I/O completion ports / worker threads; recursive natively; rename pairs and buffer overflows are the main quirks.
- **`PollingObserver`** — no OS support required. It periodically snapshots directory trees and diffs them. Portable and correct on network/exotic filesystems, but O(files) work every interval and inherently latent.

`watchmedo` builds on the library: it maps CLI flags (`--patterns`, `--recursive`, `--ignore-directories`) onto `PatternMatchingEventHandler` and provides subcommands (`log`, `shell-command`, `auto-restart`, `tricks-from`). "Tricks" are `watchdog.tricks.Trick` subclasses declared in a `tricks.yaml`, letting third parties ship reusable event handlers configured entirely in YAML.

## Production Notes

- **inotify watch exhaustion is the #1 operational failure.** On Linux, recursively watching a large tree (a `node_modules`, a monorepo) can hit `fs.inotify.max_user_watches` (often 8192 or 65536 by default). Symptoms are silently missed events or an `OSError: inotify watch limit reached`. Fix by raising the sysctl (`fs.inotify.max_user_watches`) or by excluding large subtrees — watchdog has no built-in ignore-during-scan for this.
- **Atomic saves defeat modify events.** Editors like Vim (and many "safe write" flows) write a temp file and rename it over the target. Watchdog sees create/move/delete on the temp path, not a modify on the original. If you need "file X changed," watch for moves into the path, not just `on_modified`.
- **Network and userspace filesystems need polling.** CIFS/SMB, some NFS mounts, and certain FUSE/overlay filesystems do not deliver native change notifications. Explicitly use `from watchdog.observers.polling import PollingObserver as Observer` — the auto-selected native observer will appear to work but silently drop events.
- **Duplicate and coalesced events are expected, not bugs.** The same logical change can surface as multiple events, and rapid changes can be coalesced into one. Consumers should debounce (e.g. collapse events within a short window per path) rather than assume 1:1 fidelity.
- **Handlers run on the dispatch thread.** Long-running or blocking callbacks delay all subsequent events. Hand work off to a queue/worker if processing is nontrivial.
- **kqueue FD limits on BSD/macOS.** If you force kqueue or run on FreeBSD, raise `ulimit -n` above the number of files in the watched tree.
- **Free-threaded CPython is unaudited.** watchdog builds and runs under free-threaded (no-GIL) CPython, but the maintainers note a full thread-safety audit has not been done, especially for the macOS FSEvents path[^2]. Treat it as experimental there.

## When to Use / When Not

**Use when:**
- You need cross-platform file watching in Python with one API and don't want to write inotify/FSEvents/kqueue code yourself.
- You're building a dev-server reloader, a live-rebuild tool, or a "run this on change" automation and `watchmedo` covers it without code.
- You watch a bounded set of directories and can tolerate debouncing duplicate events.

**Avoid when:**
- You need microsecond-accurate, guaranteed-lossless event streams — no userspace library over these OS APIs gives that; consider talking to inotify/FSEvents directly.
- Your hot path is watching hundreds of thousands of files with low latency — a Rust-backed watcher (watchfiles) or a daemon like Watchman will use less CPU.
- You're only on Linux and want the thinnest possible dependency — raw `inotify` bindings may suffice.

## Alternatives

- samuelcolvin/watchfiles — Rust-backed (uses the `notify` crate), simpler API, notably lower CPU under heavy change; used by modern `uvicorn --reload`. Use instead when performance and a small event surface matter more than watchdog's tricks/CLI.
- facebook/watchman — a persistent C++ daemon with a query language, built for very large monorepos. Use when you need shared, cross-process watching at scale.
- seb-m/pyinotify — Linux-only inotify bindings. Use when you're Linux-exclusive and want direct control without the cross-platform abstraction.
- notify-rs/notify — the Rust library watchfiles wraps; use when your project is Rust, not Python.
- inotify-tools/inotify-tools — `inotifywait`/`inotifywatch` shell utilities. Use for quick Linux shell scripting without Python.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010 | Started by Yesudeep Mangalapilly at Google[^1]. |
| 0.8.x | ~2014 | Maintenance moves to Thomas Amland & contributors. |
| 1.0.0 | 2020-11 | Python 2 support dropped; Python 3-only line begins. |
| 2.0.0 | 2021-02 | Type hints, API cleanup, event-handling refinements. |
| 3.0.0 | 2023 | Older Python versions dropped; internal modernization. |
| 4.0.0 | 2024 | Continued cleanup; minimum Python raised. |
| 6.0.0 | 2024 | Current major line; Python 3.9+ required[^3]. |

## References

[^1]: Copyright / maintainership history from the project README ("Copyright 2011–2012 Yesudeep Mangalapilly", "2012–2014 Google, Inc.", "2018–2025 Mickaël Schoentgen"). https://github.com/gorakhargosh/watchdog#licensing
[^2]: README, "Free threaded support" section — notes a full thread-safety audit is not complete, particularly the macOS FSEvents interface. https://github.com/gorakhargosh/watchdog
[^3]: README states "Works on 3.9+"; version/date details are in the project changelog. https://github.com/gorakhargosh/watchdog/blob/master/changelog.rst

## Tags

python, filesystem, file-watching, inotify, fsevents, kqueue, cross-platform, events, cli, developer-tools
