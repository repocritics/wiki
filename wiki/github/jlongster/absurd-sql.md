# jlongster/absurd-sql

> Persistent SQLite in the browser by treating IndexedDB as a block device —
> a landmark hack now largely superseded by SQLite's official WASM/OPFS build.

[GitHub repo](https://github.com/jlongster/absurd-sql) ·
[License: MIT](https://github.com/jlongster/absurd-sql/blob/master/LICENSE.txt)

## Overview

absurd-sql is an IndexedDB-backed filesystem for sql.js (SQLite compiled to
WebAssembly), written by James Long (author of Prettier and Actual Budget)
and launched in mid-2021 alongside the essay "A future for SQL on the
web"[^1]. Before it, persisting sql.js meant serializing the whole database
image on every save. absurd-sql instead stores the database as page-sized
blocks inside IndexedDB and reads/writes only the blocks SQLite touches —
storing "a whole database into another database. Which is absurd," per the
README[^2]. The result was incremental, transactional SQLite persistence in
the browser, benchmarked at up to ~10x faster than equivalent raw IndexedDB
access patterns[^2].

The project's defining tension is stated in its own GitHub description:
"sqlite3 in ur indexeddb (hopefully a better backend soon)." It was always a
bridge technology, and the better backend arrived: SQLite 3.40 (November
2022) shipped an officially supported WASM build with an OPFS (Origin Private
File System) VFS[^3], removing the need to route file I/O through IndexedDB
at all. The last commit landed in August 2023; with 41 open issues against a
dormant master branch, absurd-sql is best read today as prior art and a
reference implementation, not a library to adopt. Its 4.3k stars reflect
historical influence — it proved serious local-first SQL apps (most notably
Actual Budget, which shipped on it[^4]) were viable on the web platform.

## Getting Started

Requires the author's fork of sql.js and must run inside a Web Worker:

```bash
yarn add @jlongster/sql.js absurd-sql
```

```js
// index.worker.js
import initSqlJs from '@jlongster/sql.js';
import { SQLiteFS } from 'absurd-sql';
import IndexedDBBackend from 'absurd-sql/dist/indexeddb-backend';

async function run() {
  let SQL = await initSqlJs({ locateFile: file => file });
  let sqlFS = new SQLiteFS(SQL.FS, new IndexedDBBackend());
  SQL.register_for_idb(sqlFS);

  SQL.FS.mkdir('/sql');
  SQL.FS.mount(sqlFS, {}, '/sql');

  let db = new SQL.Database('/sql/db.sqlite', { filename: true });
  db.exec(`PRAGMA journal_mode=MEMORY;`);
  db.exec(`CREATE TABLE IF NOT EXISTS kv (key TEXT, value INTEGER)`);
}
run();
```

The main thread must call `initBackend(worker)` from
`absurd-sql/dist/indexeddb-main-thread` (it also proxies worker creation for
Safari, which lacked nested workers)[^2]. The server must send
`Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy:
require-corp`, because `SharedArrayBuffer` only exists in cross-origin
isolated contexts.

## Architecture / How It Works

The core problem: SQLite's C code performs synchronous file reads, but every
IndexedDB API is asynchronous. absurd-sql resolves this with
`SharedArrayBuffer` + `Atomics`. SQLite runs in one worker; IndexedDB
operations are serviced elsewhere; when SQLite requests a block, the calling
worker parks on `Atomics.wait` until the bytes are written into the shared
buffer. This makes async storage look synchronous without ever blocking the
main thread[^1].

On top of that sits `SQLiteFS`, an Emscripten filesystem registered with
sql.js (why the `@jlongster/sql.js` fork is required — upstream lacked the
registration hook). Database files are split into page-aligned blocks stored
as IndexedDB records. A key insight from the launch essay is transaction
reuse: opening a fresh IndexedDB transaction per read is what makes
IndexedDB slow, so absurd-sql holds long-lived transactions and streams
reads through cursors where profitable — the source of most of its win over
naive IndexedDB usage[^1]. Because SQLite's own journal doubles write
traffic through this already indirect path, the README recommends `PRAGMA
journal_mode=MEMORY`, delegating atomicity to IndexedDB's transactional
writes instead[^2].

**Fallback mode** (browsers without `SharedArrayBuffer`, i.e. Safari at the
time): reads are pre-loaded via `readIfFallback()`, and only one tab may
write at a time — concurrent writers get an error, not corruption[^2].

## Production Notes

- **Unmaintained.** No commits since 2023-08; 41 open issues. Bugs in the
  block layer or against newer browser releases will not be fixed.
- **COOP/COEP is invasive.** Cross-origin isolation breaks embedding of
  non-CORP third-party resources (iframes, images, scripts). Retrofitting
  these headers is often the largest adoption cost, and it applies to the
  whole document, not just the database code.
- **Forked dependency.** `@jlongster/sql.js` tracks an old sql.js/SQLite
  vintage — no current SQLite features or upstream security fixes.
- **Double-database overhead.** Every byte passes through IndexedDB's own
  storage machinery. It beats naive IndexedDB usage, but OPFS-based VFSes
  with more direct file access are the faster architecture on current
  browsers.
- **Multi-tab writes** are safe but coarse: with SharedArrayBuffer, locking
  works; in fallback mode, a second writing tab simply throws.
- **Storage eviction** applies: IndexedDB lives under normal browser quota
  rules and can be evicted unless the origin holds persistent-storage
  permission. Treat the local database as a cache or pair it with sync.

## When to Use / When Not

**Use when:**
- You maintain an existing absurd-sql app (e.g., an Actual Budget-era
  codebase) and need to understand its storage layer.
- You need SQLite persistence on old browser targets where OPFS is
  unavailable but SharedArrayBuffer is.
- You are studying the SharedArrayBuffer/Atomics sync-over-async pattern —
  the code and launch essay remain a strong reference.

**Avoid when:**
- Starting any new project: use SQLite's official WASM build with OPFS, or
  wa-sqlite, both actively maintained.
- You cannot ship COOP/COEP headers (pages embedding third-party widgets/ads).
- Your data is key-value or document-shaped — an IndexedDB wrapper is less
  machinery than SQL-in-WASM.

## Alternatives

- sqlite/sqlite-wasm — official SQLite WASM distribution with OPFS VFS; the
  default choice for new browser-SQLite projects since late 2022.
- rhashimoto/wa-sqlite — maintained WASM SQLite with pluggable VFSes
  (IndexedDB and OPFS variants); use when you need backend flexibility or
  can't require cross-origin isolation everywhere.
- sql-js/sql.js — SQLite in WASM, no persistence layer; fine for in-memory
  analysis of uploaded files.
- electric-sql/pglite — Postgres-in-WASM with IndexedDB/OPFS persistence;
  use when you want Postgres semantics client-side.
- dexie/Dexie.js — ergonomic IndexedDB wrapper; use when you don't need SQL.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021-07 | Repository created. |
| 0.0.x | 2021-08 | Public launch with "A future for SQL on the web" essay; demo + Actual Budget preview[^1]. |
| — | 2022-04 | Actual Budget, the primary production consumer, open-sourced[^4]. |
| — | 2022-11 | SQLite 3.40 ships official WASM + OPFS support — the "better backend" the repo description anticipated[^3]. |
| — | 2023-08 | Last commit; project dormant since. |

## References

[^1]: James Long, "A future for SQL on the web" — 2021-08. https://jlongster.com/future-sql-web
[^2]: absurd-sql README. https://github.com/jlongster/absurd-sql
[^3]: SQLite 3.40.0 release notes (WASM/JS officially supported, OPFS VFS) — 2022-11-16. https://sqlite.org/releaselog/3_40_0.html
[^4]: Actual Budget (open-source local-first budgeting app built on absurd-sql). https://github.com/actualbudget/actual

## Tags

javascript, sqlite, wasm, indexeddb, browser-storage, local-first, offline-first, web-workers, sql, persistence
