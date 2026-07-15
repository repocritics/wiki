# paulmillr/chokidar

> Cross-platform file watching for Node.js that normalizes the mess of `fs.watch` / `fs.watchFile` into reliable add/change/unlink events.

[GitHub repo](https://github.com/paulmillr/chokidar) ·
[Author site](https://paulmillr.com) ·
[License: MIT](https://github.com/paulmillr/chokidar/blob/main/LICENSE)

## Overview

Chokidar is the file-watching layer that most of the JavaScript toolchain quietly runs on. It was extracted from the Brunch build tool in 2012[^1] and now sits under an enormous share of the ecosystem — the README claims use in roughly 30 million repositories, transitively via webpack, Vite (historically), nodemon, browser-sync, the VS Code extension host, PostCSS/Tailwind watch modes, and countless dev servers. If a Node tool has a `--watch` flag, chokidar is the odds-on implementation behind it.

The reason it exists is that Node's built-in watching primitives are inconsistent and, on their own, close to unusable for real tooling. `fs.watch` reports platform-specific event names (`rename` for almost everything), sometimes omits filenames, double-fires, and behaves differently on macOS, Linux, and Windows. `fs.watchFile` is portable but polls, burning CPU. Chokidar's job is to paper over all of that: pick a sensible backend per platform, `stat()` the filesystem to figure out what actually happened, and emit a clean, deduplicated stream of semantic events (`add`, `addDir`, `change`, `unlink`, `unlinkDir`).

The defining tension of the project is *fidelity versus resource cost*. Getting reliable events means extra `stat()` calls, recursive watcher setup, and sometimes falling back to polling — all of which cost file descriptors and CPU. Chokidar's history (see below) is largely a story of shrinking that overhead: v3 cut CPU/RAM dramatically, and v4 removed glob support and the bundled native `fsevents` dependency to collapse the dependency tree from 13 packages to 1[^2].

## Getting Started

```sh
npm install chokidar
```

```javascript
import chokidar from 'chokidar';

// One-liner: watch the current directory
chokidar.watch('.').on('all', (event, path) => {
  console.log(event, path);
});

// Explicit options + typed events
const watcher = chokidar.watch('src', {
  ignored: (path, stats) => stats?.isFile() && !path.endsWith('.js'),
  ignoreInitial: true,      // skip the initial "add" flood on startup
  persistent: true,
});

watcher
  .on('add',     p => console.log(`added   ${p}`))
  .on('change',  p => console.log(`changed ${p}`))
  .on('unlink',  p => console.log(`removed ${p}`))
  .on('ready',   () => console.log('initial scan complete'));

// close() is async — await it to avoid leaked watchers
await watcher.close();
```

Note the v4 API change: `ignored` now takes a function/regex/path, **not** a glob string. If you previously watched `'**/*.js'`, you filter with a predicate or pre-expand paths via Node's built-in `fs.glob`[^2].

## Architecture / How It Works

Chokidar does not implement its own kernel-level watching — it orchestrates Node's `fs` primitives and normalizes them.

- **Backend selection.** The default is `fs.watch` (event-driven, low CPU). On network filesystems, some virtualized environments, and other cases where `fs.watch` misfires, you opt into `fs.watchFile` polling via `usePolling: true` (or the `CHOKIDAR_USEPOLLING` env var). Polling is reliable everywhere but scales with the number of files, not the number of changes.
- **Event normalization.** Because `fs.watch` events are unreliable and often just say `rename`, chokidar responds to a raw event by re-`stat()`-ing the path and/or re-reading the directory to determine what actually changed, then emits the correct semantic event. This is why chokidar is trustworthy where raw `fs.watch` is not — and why it does more syscalls than the raw API.
- **Recursive watching.** Chokidar always watches recursively within the supplied paths (with an optional `depth` cap), setting up a watcher per directory in scope. This is the main driver of file-descriptor consumption on large trees.
- **Atomic writes.** Many editors save by writing a temp file and renaming over the target, which naively looks like unlink-then-add. The `atomic` option (on by default when not using polling) collapses a delete + re-add within ~100 ms into a single `change` event.
- **Chunked writes.** `awaitWriteFinish` polls file size and holds `add`/`change` until the size stops changing for `stabilityThreshold` ms — the standard fix for reacting to large files before the writer has finished.

As of v4 the library is TypeScript, ships its own types, and has exactly one runtime dependency (`readdirp`, also by the author). Native macOS `fsevents` is no longer bundled; chokidar uses `fs.watch` on macOS by default[^2].

## Production Notes

- **`ENOSPC` / `EMFILE` on Linux is the classic failure.** Recursive watching consumes inotify watches; on large trees you hit `Error: ENOSPC: System limit for number of file watchers reached`. The fix is OS-level, not library-level: raise `fs.inotify.max_user_watches` (e.g. `echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p`). Watch narrowly and cap `depth` to avoid it.
- **Polling is a CPU tax that scales with file count.** `usePolling: true` and a low `interval` will keep a laptop fan running on a large repo. It is often *required* to watch files on network shares, Docker-on-Mac/Windows bind mounts, and some VMs — where `fs.watch` events never arrive — so many teams are forced into it and then must tune `interval` / `binaryInterval` to trade latency against CPU.
- **`close()` is async.** Forgetting to `await watcher.close()` leaks watchers and file descriptors across test runs and hot-reload cycles — a common source of "too many open files" in CI.
- **The initial scan emits an event per existing file.** Without `ignoreInitial: true`, a watcher over a big tree fires a burst of `add` events on startup before `ready`. Tools that treat every `add` as "rebuild" will do a spurious full rebuild at launch.
- **v4/v5 dropped globs — this is the top upgrade friction.** Code passing `'**/*.ext'` to `watch()` silently stops matching; you must migrate to the `ignored` predicate form or pre-expand paths. Budget for this when upgrading from v3.
- **fsevents warnings are gone in v4+.** The long-standing `npm WARN optional dep failed` and `fsevents is not a constructor` noise on non-macOS installs was a v3-and-earlier artifact of the bundled native dep; upgrading to v4+ removes it.
- **v5 is ESM-only.** As of the November 2025 release, chokidar 5 dropped CommonJS and raised the floor to Node.js v20[^3]. CommonJS codebases that cannot `import()` should pin to v4.

## When to Use / When Not

**Use when:**
- You need reliable, semantic add/change/unlink events across macOS, Linux, and Windows without writing per-platform glue.
- You're building dev tooling, a build watcher, a live-reload server, or a config hot-reloader.
- You need atomic-write and chunked-write handling (editors, large-file pipelines) out of the box.

**Avoid / reconsider when:**
- You're on Node 19+ and only need to watch a single directory recursively with basic events — the built-in `fs.watch(dir, { recursive: true })` may suffice with zero dependencies.
- You're watching enormous trees where inotify limits and descriptor exhaustion dominate — consider a watcher daemon like Watchman that is built for scale.
- You need cross-language or out-of-process watching — chokidar is Node-only.

## Alternatives

- facebook/watchman — a standalone watching service (C++), built for very large repos and multi-language clients; heavier to deploy but scales past inotify pain.
- Node.js core `fs.watch` — zero dependencies, recursive mode on modern Node; use it when you don't need chokidar's event normalization or atomic/chunked handling.
- parcel-bundler/watcher (`@parcel/watcher`) — native (N-API) watcher with an event-subscription model and native macOS/Windows backends; use when you want native performance and don't need chokidar's pure-JS portability.
- gulpjs/chokidar-cli / open-cli-tools/chokidar-cli — thin CLI wrappers around chokidar; use when you want run-command-on-change from a shell rather than in code.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2012-04 | Initial release, extracted from Brunch[^1]. |
| 1.0 | 2015-04 | Glob support, symlink support; Node 0.8+. |
| 2.0 | 2017-12 | POSIX-style globs only; large bugfix pass. |
| 3.0 | 2019-04 | Major CPU/RAM cuts; ~17x smaller deps/size; Node 8.16+. |
| 4.0 | 2024-09 | Rewrite in TypeScript; glob support and bundled fsevents removed; deps 13 → 1; Node 14+[^2]. |
| 5.0 | 2025-11 | ESM-only; minimum Node.js raised to v20[^3]. |

Repository stats at time of writing: ~12.2k stars, ~631 forks, MIT-licensed, created April 2012, last pushed July 2026 — long-lived and still actively maintained by a single primary author (Paul Miller).

## References

[^1]: Chokidar README — "Made for Brunch in 2012," origin and design rationale. https://github.com/paulmillr/chokidar#readme
[^2]: Chokidar README changelog — v4 (Sep 2024): TypeScript rewrite, glob and bundled fsevents removed, dependency count reduced from 13 to 1. https://github.com/paulmillr/chokidar#changelog
[^3]: Chokidar README — "Nov 2025 update: v5 is out. Makes package ESM-only and increases minimum node.js requirement to v20." https://github.com/paulmillr/chokidar#readme

## Tags

javascript, typescript, nodejs, file-watching, filesystem, fsevents, inotify, developer-tooling, build-tools, cross-platform
