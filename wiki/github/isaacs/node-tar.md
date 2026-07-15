# isaacs/node-tar

> A full-featured, security-hardened tar implementation in pure JavaScript — the one npm itself uses to pack and unpack packages.

[GitHub repo](https://github.com/isaacs/node-tar) ·
[License: BlueOak-1.0.0](https://github.com/isaacs/node-tar/blob/main/LICENSE.md)

## Overview

node-tar is a streaming implementation of the Unix `tar` archive format written in JavaScript, by Isaac Z. Schlueter (isaacs), the original author of npm[^1]. It is the library npm uses to create package tarballs on `npm publish` and to unpack them on `npm install`, which makes it one of the most-executed pieces of code in the JavaScript ecosystem despite a modest star count — install traffic, not GitHub stars, is the real measure of its footprint. The repository dates back to 2011 and remains actively maintained, with a push to `main` within the last few days as of mid-2026.

The API deliberately mirrors the `tar(1)` command line: the five top-level operations expose both single-character names (`c`, `r`, `u`, `t`, `x`) for people who know Unix tar and long aliases (`create`, `replace`, `update`, `list`, `extract`) for everyone else[^2]. Options map onto familiar flags — `z` for gzip, `C` for cwd, `strip` for `--strip-components`.

The defining tension of this project is **security versus fidelity**. Extraction of an archive from an untrusted source is a notorious source of path-traversal and symlink attacks, and node-tar carries years of hardening against them. But hardening is not the same as safety: the maintainer is explicit that some attacks (TOCTOU races when writing to an attacker-controlled directory) are unavoidable in principle, and that responsibility for using the library safely still rests with the caller[^3]. This is the rare infrastructure README that tells you, in so many words, that you can vibe-code a tar extractor in an afternoon and you will regret it.

## Getting Started

```bash
npm install tar
```

```js
import * as tar from 'tar'

// Create a gzipped archive (tar czf backup.tgz ./src ./package.json)
await tar.c(
  { gzip: true, file: 'backup.tgz' },
  ['src', 'package.json'],
)

// Extract into ./out, stripping the leading path component
await tar.x({
  file: 'backup.tgz',
  cwd: 'out',
  strip: 1,
})

// List entries without touching disk
await tar.t({
  file: 'backup.tgz',
  onReadEntry: entry => console.log(entry.path, entry.size),
})
```

If no `file` option is given, the same functions return streams instead: `tar.c(...)` yields a readable stream of archive bytes, and `tar.x()` / `tar.t()` yield writable streams you pipe an archive into.

## Architecture / How It Works

node-tar is built as two layers. The **high-level API** (`c`/`r`/`u`/`t`/`x`) is a thin convenience wrapper that assembles the right streams, wires up gzip detection, and handles the file-vs-stream and sync-vs-async permutations. Underneath sits a **low-level streaming API**: `Pack` and `Unpack` (writable streams), `Parse` (a transform that turns bytes into a sequence of entry objects), and `ReadEntry` / `WriteEntry` classes representing individual archive members. The high-level functions are entirely expressible in terms of the low-level ones, and the library exposes both.

Everything is stream-first. An archive is processed as it arrives rather than buffered whole, which is what lets npm unpack multi-hundred-megabyte tarballs without proportional memory use. Gzip (and, in recent versions, other compressions) is detected from the byte stream and decompressed inline. Each operation has a synchronous variant triggered by `sync: true`, which runs to completion in the same tick and returns no promise or callback.

The security machinery lives almost entirely in the extraction path (`Unpack`)[^3]:

- Entries whose paths walk outside the extraction target (`..`) are skipped with a `warn`, not extracted.
- `Link` and `SymbolicLink` entries may not point outside the target, and extraction is refused *through* a symlink that already exists inside the target.
- Absolute paths are rewritten to relative; character/block devices and FIFOs are never created.
- A **path-reservation system** serializes concurrent entries that touch the same filename, closing a race where a file could be swapped for a symlink mid-write.
- Unicode path names are normalized so equivalence tricks cannot bypass the checks.

Almost all of these protections are disabled when `preservePaths: true` is set — a deliberate escape hatch for trusted archives that also removes the safety net entirely.

Errors are surfaced through a two-tier warning system. Recoverable problems emit a `warn` event with a tar-specific `code` (`TAR_ENTRY_INVALID`, `TAR_ENTRY_INFO`, `TAR_ENTRY_UNSUPPORTED`, `TAR_BAD_ARCHIVE`, and so on) and are only thrown as errors when `strict: true` is set; unrecoverable ones always throw or reject regardless. Errors bubbling up from `fs` or `zlib` keep their original `code` and get a `tarCode` field attached, so callers should read `error.tarCode` to see how tar itself classified the failure.

## Production Notes

**The security warnings are load-bearing, not boilerplate.** The single most important operational rule is: never extract an untrusted tarball into a directory that an attacker could also write to. No amount of library hardening closes the TOCTOU window there, and the maintainer states outright that security reports about this class of attack will be closed[^3]. If you unpack third-party archives, add a `filter` function that rejects all hardlinks and symbolic links — link entries are the historical root of nearly every tar extraction CVE, and npm strips them from package artifacts for exactly this reason.

**Decompression bombs are your problem, not tar's.** A small compressed archive can expand to fill a disk. If you accept compressed input from users, filter on entry size during extraction rather than trusting the archive's on-disk footprint.

**node-tar has a real CVE history**, which is a feature (it means the hardening was tested by fire) but also a reason to stay current. A cluster of high-severity path-traversal and arbitrary-file-write advisories landed in 2021 and were patched across the then-supported release lines[^4]. The README's fourth safety rule is blunt: old versions should be assumed to contain every known vulnerability. Pin a maintained major and keep it patched.

**Version discontinuities are the main upgrade pain.** Major releases have periodically dropped older Node versions and reworked internals. Version 7 (2024) was rewritten in TypeScript and ships as a hybrid ESM/CommonJS package requiring a modern Node runtime[^5]; treat a major-version bump as a real migration, check the `mode`/ownership defaults (files are extracted with archive modes ignored unless `chmod: true`, and ownership is not changed unless running as root or `forceChown` is set), and re-read the options list rather than assuming names carried over.

**Modes and ownership do not round-trip by default.** On creation, `portable: true` strips system-specific metadata (uid/gid/ctime/atime/etc.) for reproducible archives. On extraction, archive file modes are ignored unless you opt in. Teams expecting `tar` CLI parity are sometimes surprised that permissions and owners come out normalized.

## When to Use / When Not

**Use when:**
- You need to create or extract `.tar` / `.tar.gz` archives in Node with a battle-tested, security-conscious implementation.
- You are handling archives from untrusted sources and want extraction hardening you did not have to write yourself.
- You want an API that maps cleanly onto `tar(1)` semantics and streams rather than buffering.

**Avoid when:**
- You need the ZIP format — node-tar does not do ZIP; reach for a dedicated library.
- You want a minimal dependency and only need to parse/produce raw tar streams with no filesystem or security layer — a lower-level module is lighter.
- You are shelling out to a guaranteed-present system `tar` already and don't need a portable pure-JS path.

## Alternatives

- mafintosh/tar-stream — low-level streaming tar parser/packer with no filesystem or security layer; use it when you want raw control over entries and will handle safety yourself.
- mafintosh/tar-fs — filesystem-oriented pack/extract with a simpler API and fewer built-in protections; use it for archiving trusted local directories where node-tar's hardening is not the priority.
- archiverjs/node-archiver — high-level archive creation for both tar and ZIP output; use it when you primarily generate archives (not extract untrusted ones) and want one API across formats.
- system `tar` via `child_process` — use it when a maintained `tar(1)` is guaranteed present and you want its exact CLI behavior without a JS dependency.
- adm-zip / yauzl — use these instead when the format you actually need is ZIP, not tar.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2011 | First published; pure-JS tar for early Node/npm. |
| 6.x | ~2019–2020 | Major rework; dropped older Node versions, streaming internals modernized. |
| — | 2021 | Cluster of path-traversal / arbitrary-file-write CVEs patched across supported lines[^4]. |
| 7.0 | 2024 | Rewritten in TypeScript; hybrid ESM/CJS package; requires a modern Node runtime[^5]. |

## References

[^1]: node-tar repository and README, isaacs/node-tar. https://github.com/isaacs/node-tar
[^2]: node-tar README, "High-Level API" — command name aliases mirroring `tar(1)`. https://github.com/isaacs/node-tar#high-level-api
[^3]: node-tar README, "Security Information" — hardening measures, `preservePaths` escape hatch, and the TOCTOU caveat. https://github.com/isaacs/node-tar#security-information
[^4]: GitHub Advisory Database, node-tar (npm package `tar`) advisories. https://github.com/advisories?query=node-tar
[^5]: node-tar package on npm (release/version history). https://www.npmjs.com/package/tar

## Tags

javascript, nodejs, tar, archive, compression, streaming, gzip, npm, filesystem, security, cli-tooling
