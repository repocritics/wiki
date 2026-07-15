# jprichardson/node-fs-extra

> A drop-in `fs` replacement that adds the recursive filesystem operations Node.js left out — `copy`, `remove`, `move`, `mkdirs` — plus promises on everything.

[GitHub repo](https://github.com/jprichardson/node-fs-extra) ·
[License: MIT](https://github.com/jprichardson/node-fs-extra/blob/master/LICENSE)

## Overview

`fs-extra` began in 2011 as one person's fatigue with copy-pasting `mkdirp`, `rimraf`, and `ncp` into every project[^1]. It is a thin superset of Node's built-in `fs`: it re-exports every native method and adds the higher-level operations that the standard library historically refused to ship — recursive copy, recursive delete, recursive mkdir, atomic-ish JSON read/write, and "write this file and create any missing parent directories" helpers. Because it forwards all of `fs`, you can replace `require('fs')` with `require('fs-extra')` and lose nothing.

The library sits at the very bottom of the JavaScript dependency graph. It is one of the most-depended-upon npm packages in existence, pulled in transitively by build tools, scaffolders, and CLIs across the ecosystem. That position is the whole story of the project: it is deliberately boring, changes rarely, and treats a breaking change as a serious event because millions of installs feel it.

The defining tension is that Node's own `fs` has been catching up. Since Node 10 the native module exposes `fs.promises`; since Node 16 `fs.rm({ recursive: true })` and `fs.cp({ recursive: true })` cover the two headline features (`remove` and `copy`)[^2]. `fs-extra` still earns its place — `outputFile`, `ensureDir`, `readJson`/`writeJson`, `move` across devices, `graceful-fs` EMFILE handling — but a new project that only needs recursive copy/delete no longer strictly requires it.

## Getting Started

```bash
npm install fs-extra
```

```js
const fs = require('fs-extra')

// recursive copy — no external ncp needed
await fs.copy('/tmp/src', '/tmp/dst')

// write a file, creating any missing parent dirs
await fs.outputFile('/tmp/a/b/c/hello.txt', 'hi')

// remove a file or a whole tree, no error if it's already gone
await fs.remove('/tmp/a')

// JSON round-trip with pretty-printing
await fs.writeJson('/tmp/pkg.json', { name: 'x' }, { spaces: 2 })
const data = await fs.readJson('/tmp/pkg.json')
```

Every async method returns a promise when no callback is passed, and each has a `*Sync` twin (`copySync`, `removeSync`, …) that throws on error. An `fs-extra/esm` entry point exports the extra methods as named ESM exports, but it deliberately does not re-export native `fs` — under ESM you import `fs`/`fs/promises` yourself[^3].

## Architecture / How It Works

The package is intentionally shallow. At load time it builds its export object by copying every method off native `fs`, promisifying the callback-style ones via a universalify wrapper so they work with both callbacks and `await`, and layering the extra methods on top. The extras are small modules composed from primitives: `outputFile` is `ensureDir(dirname) → writeFile`; `ensureDir` is essentially `mkdir({ recursive: true })`; `move` is a `rename` that falls back to copy-then-delete when the source and destination live on different devices (the `EXDEV` case).

Two hard dependencies do the load-bearing work. `graceful-fs`[^4] patches `fs` to queue and retry operations that fail with `EMFILE`/`ENFILE` (too many open file descriptors), which is why `fs-extra` survives copying thousands of files where naive `fs` code would crash. `jsonfile`[^5] — also by the same author — implements the JSON read/write layer with BOM stripping and configurable spacing.

`copy` is the most complex method and the one that carries most of the project's edge-case history: it has to reproduce file modes, decide how to treat symlinks, refuse to copy a directory into itself, optionally preserve timestamps, and honor a `filter` predicate. The Node core `fs.cp` that arrived in Node 16 is essentially the standard library absorbing this logic, and its early releases replicated several `fs-extra` bugs before both converged.

Notably absent: `walk`/`walkSync` were removed in v2.0.0 (2017) and spun out into the separate `klaw` and `klaw-sync` packages[^6]. `fs-extra` has resisted growing a traversal API ever since — a good example of its scope discipline.

## Production Notes

- **It is `fs` semantics, not magic.** `copy` and `remove` are not atomic. A crash mid-copy leaves a partial tree; a `remove` on a tree that another process is writing to can race. For atomic file replacement, write to a temp path and `rename`.
- **`move` across devices is copy-then-delete.** On the same filesystem it is a cheap `rename`; across a device boundary (`EXDEV` — e.g. Docker volume to overlay, or `/tmp` tmpfs to disk) it silently degrades to a full recursive copy followed by delete, which is far slower and briefly doubles disk usage. The test suite even gates cross-device tests behind a `CROSS_DEVICE_PATH` env var because the behavior is easy to miss[^3].
- **Symlink handling in `copy` has real subtlety.** By default symlinks are copied as symlinks (dereference off). The `dereference` option changes this, and mismatches here are a recurring source of surprising output trees. Read the `copy` docs before relying on the default.
- **Windows symlinks need elevation.** `ensureSymlink` and symlink-related tests throw `EPERM` on Windows unless the process has the privilege to create symbolic links — a frequent CI failure for cross-platform projects[^3].
- **Native `fs` may already be enough.** If your only needs are recursive copy and recursive delete on Node 16+, `fs.cp`/`fs.rm` remove the dependency. Keep `fs-extra` when you actually use `outputFile`, `ensureDir`, the JSON helpers, cross-device `move`, or want `graceful-fs`'s EMFILE protection under heavy concurrency.
- **Deprecated `fs` constants.** As of Node 24, `fs.F_OK`/`R_OK`/`W_OK`/`X_OK` are no longer exported through `fs` (and therefore not through `fs-extra`); use `fs.constants.*`[^3].
- **Stability as a feature.** Releases are infrequent and conservative. This is desirable at this layer, but it also means the package moves slowly relative to new Node APIs; do not expect same-cycle wrappers for the newest `fs` additions.

## When to Use / When Not

**Use when:**
- You want `fs` plus recursive copy/move/remove and "write-and-create-parents" with one import and no glue.
- You do frequent JSON file I/O and want `readJson`/`writeJson`/`outputJson` instead of hand-rolled `JSON.parse(readFile(...))`.
- You copy or open many files concurrently and want `graceful-fs`'s EMFILE retry behavior for free.
- You need cross-device `move` semantics handled for you.

**Avoid when:**
- You target Node 16+ and only need recursive copy/delete — native `fs.cp`/`fs.rm` cover it dependency-free.
- You need directory traversal — reach for `klaw`/`klaw-sync` or `fast-glob` instead; `fs-extra` deliberately omits it.
- You need atomic or transactional file operations — use a dedicated approach (temp file + `rename`, or `write-file-atomic`).

## Alternatives

- `sindresorhus/del` — focused, glob-aware recursive delete; use instead when you only need robust `rm -rf` with pattern matching.
- `isaacs/rimraf` — the standalone recursive-delete workhorse; use when you want just `remove` with battle-tested Windows retry logic.
- `isaacs/node-mkdirp` — standalone recursive mkdir; use when `ensureDir` is the only extra you need (or just use native `fs.mkdir({recursive:true})`).
- `jprichardson/node-klaw` — the traversal API split out of fs-extra; use when you need to walk a tree.
- Node's built-in `fs` (`fs.cp`, `fs.rm`, `fs.promises`) — use when your Node version is new enough that the extras no longer justify a dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2011-11 | Initial release; collected `mkdirp`/`rimraf`/`ncp`-style helpers[^1]. |
| 1.0 | 2015-11 | First stable major. |
| 2.0 | 2017-01 | Removed `walk`/`walkSync`; split into `klaw`/`klaw-sync`[^6]. |
| 3.0 | 2017-04 | Native `fs` methods promisified and forwarded. |
| 7.0 | 2018 | Dropped legacy Node support; modernized internals. |
| 10.0 | 2021 | `fs-extra/esm` entry point for named ESM exports[^3]. |
| 11.0 | 2022 | Current major line; ongoing maintenance releases. |

## References

[^1]: fs-extra README, "Why?" — the project originated to avoid repeatedly bundling `mkdirp`, `rimraf`, and `ncp`. https://github.com/jprichardson/node-fs-extra#why
[^2]: Node.js `fs` documentation — `fs.cp`, `fs.rm`, and `fs.promises` cover much of fs-extra's original value. https://nodejs.org/api/fs.html
[^3]: fs-extra README — ESM entry point, sync/async semantics, cross-device move testing, Windows symlink `EPERM`, and Node 24 constant deprecation notes. https://github.com/jprichardson/node-fs-extra/blob/master/README.md
[^4]: graceful-fs — EMFILE/ENFILE queueing that fs-extra depends on. https://github.com/isaacs/node-graceful-fs
[^5]: jsonfile — the JSON read/write layer used by fs-extra. https://github.com/jprichardson/node-jsonfile
[^6]: klaw / klaw-sync — traversal packages extracted from fs-extra in v2.0.0. https://github.com/jprichardson/node-klaw

## Tags

javascript, nodejs, filesystem, fs, file-io, npm-package, copy, remove, json, cross-platform, utility-library
