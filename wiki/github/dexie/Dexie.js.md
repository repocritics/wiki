# dexie/Dexie.js

> A minimalistic wrapper for IndexedDB — the de facto ergonomic API for the browser's built-in database.

[GitHub repo](https://github.com/dexie/Dexie.js) ·
[Official website](https://dexie.org) ·
[License: Apache-2.0](https://github.com/dexie/Dexie.js/blob/master/LICENSE)

## Overview

Dexie.js wraps IndexedDB — the standardized, quota-managed database every browser engine ships — behind a promise-based, chainable query API. Started by David Fahlander in 2014[^1], it exists because raw IndexedDB is one of the most disliked APIs on the web platform: event-callback-driven, transaction-scoped in surprising ways, and inconsistently implemented across browsers. Dexie's pitch is not new capability but usable capability: `db.friends.where('age').below(30).toArray()` instead of cursors and `onsuccess` handlers, plus workarounds for known IndexedDB implementation bugs baked into the library.

At ~14.5k stars with a 12-year history and pushes within the past two weeks, it is the mature, conservative choice in browser storage. Since 3.2 (2021) the `liveQuery()` observable[^2] turned Dexie from a passive store into a reactive one — `useLiveQuery` in `dexie-react-hooks` re-renders components when underlying data changes, including changes made in other tabs. The defining tension: the project is the substrate for **Dexie Cloud**[^3], a commercial sync/auth service by the same author. The open-source library is genuinely standalone, but the roadmap (4.x typings, cache, `@id` keys) visibly co-evolves with the paid offering, and the older open sync addons (`dexie-observable`, `dexie-syncable`) are deprecated in its favor[^4].

## Getting Started

```bash
npm install dexie
```

```ts
import { Dexie, type EntityTable } from 'dexie';

interface Friend { id: number; name: string; age: number; }

const db = new Dexie('FriendDatabase') as Dexie & {
  friends: EntityTable<Friend, 'id'>;
};

// Schema declares only indexes, not all fields.
// '++id' = auto-increment PK, 'age' = plain index.
db.version(1).stores({ friends: '++id, age' });

await db.friends.add({ name: 'Alice', age: 21 });
const young = await db.friends.where('age').below(30).toArray();
```

`dexie-react-hooks` adds `useLiveQuery()` for React; equivalents exist for Svelte, Vue, and Angular via the `liveQuery()` observable directly[^2].

## Architecture / How It Works

**Schema-by-string.** `stores({ friends: '++id, name, [name+age]' })` declares primary key and indexes only — objects can carry arbitrary extra fields. Prefixes encode index type: `++` auto-increment, `&` unique, `*` multiEntry (index each array element), `[a+b]` compound. Schema changes require a new `version(n)` block; Dexie diffs declared versions and runs IndexedDB's `onupgradeneeded` dance, with optional `.upgrade(tx => ...)` data migrations[^5].

**The zone system is the real engineering.** IndexedDB transactions auto-commit when the microtask queue drains without a pending IDB request — historically this made native promises and `async/await` unusable inside transactions. Dexie solves it with a custom promise implementation plus "Promise-Specific Data" zones that track transaction scope across async boundaries, including through native `await`. It mostly just works, which is why nobody talks about it — until you `await fetch(...)` inside `db.transaction()` and the transaction commits underneath you (Dexie throws `PrematureCommitError`). `Dexie.waitFor()` is the documented escape hatch for keeping a transaction alive across non-IndexedDB async work, at the cost of holding locks longer[^6].

**DBCore middleware.** Dexie 3.0 (a TypeScript rewrite) introduced DBCore, a middleware stack between the public API and IndexedDB. Addons (dexie-cloud, encryption layers, the legacy hooks API) intercept reads and writes here rather than monkey-patching tables.

**liveQuery.** During query execution Dexie records which tables and key ranges were touched; mutations emit a `storagemutated` event (propagated across tabs) and only queries whose ranges intersect the mutation re-execute[^2]. Dexie 4 backs this with an in-memory cache so repeated live queries avoid hitting IndexedDB for unchanged data.

## Production Notes

**You inherit IndexedDB's failure modes.** Dexie papers over bugs but cannot change the platform:

- **Eviction.** Browser storage is best-effort by default; call `navigator.storage.persist()` and handle rejection[^7]. Safari caps script-writable storage at 7 days of non-use for non-installed web apps under ITP[^8] — a silent data-loss scenario for infrequently used tools.
- **Safari generally.** Historically the buggiest IndexedDB implementation (dropped connections after backgrounding, transaction failures). Much improved since ~14.1, but Safari-only bug reports remain a steady presence in the issue tracker.
- **Multi-tab upgrades.** A tab holding an open connection blocks another tab from opening a newer schema version. Dexie closes on `versionchange` by default, but you must handle the reload UX yourself.

**Query model, not query engine.** Only declared indexes are queryable via `where()`. `.filter()` and `.and()` are full scans in JavaScript. There is no join, no query planner beyond index selection, and compound/multiEntry indexes must be designed up front. Teams migrating from SQL habitually under-index and then wonder why queries crawl at 100k rows.

**Performance.** Every stored object pays structured-clone cost per operation. `bulkAdd`/`bulkPut` are dramatically faster than looped `put()` because they skip per-item success listeners — always use them for imports. For heavy relational or aggregate workloads, SQLite-on-OPFS approaches now outperform IndexedDB; Dexie's sweet spot is entity storage and indexed lookups, not analytics.

**Migrations are append-only.** Never edit a shipped `version(n)` block — users' databases already ran it. Add a new version; deleting a table means declaring it `null` in a later version. Changing a primary key is unsupported without copying to a new table[^5].

**Testing.** IndexedDB does not exist in Node; unit tests need `fake-indexeddb` or a real browser runner (the project itself tests in browsers).

**Sync.** The only maintained sync path is the commercial Dexie Cloud (self-hostable, but a product)[^3]. The formerly open `dexie-syncable` is explicitly deprecated[^4]. Budget for this if "offline-first with sync" is your end state and vendor coupling matters.

## When to Use / When Not

**Use when:**
- You need durable, indexed, offline client-side storage in a PWA, extension, Electron, or Capacitor app.
- You want reactive UI over local data (`liveQuery` + framework hooks) without adopting a full state-management framework.
- You need multi-tab consistency and cross-tab change propagation for free.
- Raw IndexedDB is the right platform primitive but the raw API is not.

**Avoid when:**
- You need SQL, joins, or aggregations over large datasets — use a SQLite-in-WASM approach instead.
- You only need small key-value persistence — `localStorage` or a thin wrapper is less machinery.
- You need open-protocol sync/replication as a hard requirement — RxDB or PouchDB have non-commercial sync stories.
- Your data must survive indefinitely on Safari without user installation — the 7-day ITP cap is outside any library's control[^8].

## Alternatives

- jakearchibald/idb — ~1 kB promise shim over raw IndexedDB; use it when you want the platform API with minimal abstraction and no query layer.
- pubkey/rxdb — reactive offline-first database with pluggable storage (can run on Dexie underneath) and multiple replication protocols; use it when sync flexibility matters more than simplicity.
- pouchdb/pouchdb — CouchDB replication protocol client; use it when you have CouchDB-compatible infrastructure.
- localForage/localForage — async key-value with IndexedDB/WebSQL fallback; use it when you need storage, not queries.
- sql.js / rhashimoto/wa-sqlite — SQLite compiled to WASM (OPFS-backed); use it when you need real SQL semantics in the browser.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014-02 | Project started by David Fahlander[^1]. |
| 2.0 | 2017 | Interop with native promises, improved TypeScript typings. |
| 3.0 | 2020 | TypeScript rewrite; DBCore middleware layer replaces hooks as the extension point. |
| 3.2 | 2021 | `liveQuery()` reactive queries; `dexie-react-hooks`[^2]. |
| 4.0 | 2024 | Stable after a long alpha/RC phase; `EntityTable` typings, memory cache behind liveQuery, Dexie Cloud alignment. |

## References

[^1]: GitHub repository, created 2014-02-26. https://github.com/dexie/Dexie.js
[^2]: Dexie docs, "liveQuery()". https://dexie.org/docs/liveQuery()
[^3]: Dexie Cloud — commercial sync/auth service on top of Dexie.js. https://dexie.org/cloud/
[^4]: Dexie.js README, "Legacy Addons (dexie-observable, dexie-syncable)". https://github.com/dexie/Dexie.js#legacy-addons-dexie-observable-dexie-syncable
[^5]: Dexie docs, "Database Versioning". https://dexie.org/docs/Tutorial/Design#database-versioning
[^6]: Dexie docs, "Dexie.waitFor()". https://dexie.org/docs/Dexie/Dexie.waitFor()
[^7]: MDN, `StorageManager.persist()`. https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/persist
[^8]: WebKit blog, "Full Third-Party Cookie Blocking and More" — 7-day cap on script-writable storage, 2020-03-24. https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/

## Tags

typescript, indexeddb, browser-database, offline-first, local-first, client-side-storage, reactive-queries, pwa, promises, wrapper-library
