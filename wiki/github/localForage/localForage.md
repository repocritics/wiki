# localForage/localForage

> Async browser key-value storage with a localStorage-like API, backed by
> IndexedDB, WebSQL, or localStorage — a 2014-era compatibility shim that
> outlived the problem it solved.

[GitHub repo](https://github.com/localForage/localForage) ·
[Official website](https://localforage.github.io/localForage) ·
[License: Apache-2.0](https://github.com/localForage/localForage/blob/master/LICENSE)

## Overview

localForage is a JavaScript library that gives web apps a simple asynchronous
key-value store (`getItem` / `setItem` / `removeItem` / `iterate`) and hides
the choice of backend behind a driver system: IndexedDB where available, WebSQL
as first fallback, localStorage as last resort. It started at Mozilla in
2013–2014, in the Firefox OS era, when IndexedDB support was inconsistent
across browsers and its event-based API was widely considered hostile[^1].
Unlike raw localStorage, it stores Blobs, TypedArrays, and ArrayBuffers, not
just strings, and it never blocks the main thread on the IndexedDB path.

The library's defining tradeoff has inverted over its lifetime. In 2014, "one
API over three engines" was the value; in 2026, every evergreen browser ships
a working IndexedDB, WebSQL has been removed from Chromium entirely[^2], and
Safari's IndexedDB bugs that justified the WebSQL fallback were fixed years
ago. What remains is a pleasant, tiny (~10 kB min+gzip) API veneer — and a
large installed base: 25.8k stars, 1.3k forks, and heavy npm usage via
transitive dependencies.

Maintenance status matters here: the last release is v1.10.0 (August 2021)[^3]
and the last push to the default branch was July 2024, with 249 open issues.
The API surface is frozen and stable rather than evolving; treat it as
finished software in maintenance twilight, not an active project.

## Getting Started

```bash
npm install localforage
```

```js
import localforage from "localforage";

// Optional: configure BEFORE the first data call.
localforage.config({
  name: "myApp",
  storeName: "keyvaluepairs",
});

await localforage.setItem("user", { id: 1, avatar: someBlob });
const user = await localforage.getItem("user");

await localforage.iterate((value, key, i) => {
  console.log(key, value);
});
```

The same API is available with Node-style callbacks; Promises (and therefore
`async`/`await`) are the recommended form. A UMD build works from a plain
`<script>` tag with a `localforage` global.

## Architecture / How It Works

localForage is a driver multiplexer plus a serializer:

- **Driver selection.** On first data call, drivers are probed in order
  (`asyncStorage` → `webSQLStorage` → `localStorageWrapper` by default) and
  the first supported one wins. `setDriver()` / `config({ driver })` override
  the order; `defineDriver()` registers custom backends (community drivers
  exist for cordova-sqlite, in-memory, and others).
- **Deferred readiness.** Driver probing is async, so every API call queues
  behind an internal `ready()` promise. This is why `config()` must run before
  the first `getItem`/`setItem` — after the driver initializes, database
  name/store changes are ignored for that instance.
- **Serialization.** The IndexedDB driver stores values via structured clone,
  so Blobs and TypedArrays round-trip natively. The localStorage and WebSQL
  drivers cannot, so localForage serializes binary types to marker-prefixed
  strings and JSON for everything else. The two paths are close but not
  identical — code that stores exotic objects can behave differently per
  backend.
- **Instances.** `createInstance({ name, storeName })` yields isolated stores
  (separate IndexedDB databases/object stores). `dropInstance()` (added
  1.6.0) deletes them.
- **Promise polyfill.** The library bundles `lie` for Promise support and
  carries detection code for browsers as old as IE8 — dead weight in any
  modern build, and part of why the bundle is ~10 kB rather than ~1 kB.

Each `setItem`/`getItem` opens its own IndexedDB transaction. There is no
multi-key transaction, no batching, and no query capability beyond `keys()`
and full-store `iterate()` — this is a key-value store, not a database.

## Production Notes

- **Unmaintained in practice.** No release since 2021, no commit activity
  since mid-2024. Known issues (Safari edge cases, TypeScript definition
  gaps, `iterate` typing) will not be fixed upstream. Budget for forking or
  migrating rather than waiting on patches.
- **The WebSQL driver is a liability, not a feature.** Chromium removed
  WebSQL in 2023–2024[^2]; Firefox never had it. If your config forces
  `localforage.WEBSQL`, it now silently matters only on ancient WebViews.
  Remove such config.
- **Per-key transactions are slow at scale.** Writing thousands of small
  items issues thousands of IndexedDB transactions. Community plugins
  (`localforage-setitems`, `localforage-getitems`) batch, but for bulk
  workloads a real IndexedDB wrapper (Dexie, idb) is materially faster.
- **Data eviction is the platform's, not the library's.** Safari caps
  script-writable storage at seven days of non-use under ITP[^4], and all
  browsers may evict best-effort storage under pressure. localForage predates
  `navigator.storage.persist()` and does not request persistence for you —
  call it yourself if data loss is unacceptable.
- **`null` is ambiguous.** `getItem` returns `null` both for missing keys and
  for stored `null`; `setItem(key, undefined)` stores `null`. Code that needs
  a real "key exists" check must use `keys()`.
- **Backend divergence surprises.** The same value stored under IndexedDB
  (structured clone) vs localStorage (JSON) can round-trip differently
  (e.g., `Date` objects, nested Blobs). Test on the actual fallback path if
  you support browsers that hit it.
- **Migration pain.** Data written by localForage lives in a
  localForage-shaped IndexedDB schema (`name`/`storeName` database and object
  store). Moving to Dexie or raw IndexedDB means reading via localForage and
  rewriting, or replicating its schema conventions.

## When to Use / When Not

**Use when:**

- You maintain an existing codebase already on localForage — the API is
  stable and the frozen surface is a feature for legacy code.
- You want a drop-in async replacement for a localStorage-based app with
  minimal code changes, including Blob support.
- You need a simple offline cache in a WebView matrix old enough that
  fallback drivers still earn their keep.

**Avoid when:**

- You are starting a new project in 2026 — every target browser has
  IndexedDB, and thinner or more capable wrappers exist.
- You need bulk writes, indexes, queries, or multi-key transactions —
  localForage's API cannot express them.
- You depend on active maintenance, security response, or evolving
  TypeScript support.
- Your data must survive eviction — you need `navigator.storage.persist()`
  and an eviction strategy regardless, and localForage adds nothing there.

## Alternatives

- jakearchibald/idb-keyval — use instead for new code needing only get/set:
  same job, under 1 kB, promise-native, actively maintained.
- jakearchibald/idb — use when you want real IndexedDB (indexes,
  transactions, cursors) with a thin promise layer instead of a KV facade.
- dexie/Dexie.js — use for query-heavy offline apps: schema versioning,
  compound indexes, bulk operations, live queries, optional sync.
- pouchdb/pouchdb — use when the requirement is replication/sync with
  CouchDB-compatible servers, not just local persistence.
- unjs/unstorage — use for isomorphic key-value storage with swappable
  drivers across browser, Node, and edge runtimes.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.2.0 | 2014-03 | Early Mozilla-era release; binary data support[^3]. |
| 1.0.4 | 2014-10 | "Stable API" milestone; 1.x surface essentially frozen since. |
| 1.2.10 | 2014-11 | `iterate()` added. |
| 1.5.0 | 2017-02 | Safari IndexedDB enabled (post-10.1 fixes); WebSQL demoted there. |
| 1.6.0 | 2018-03 | `dropInstance()` for deleting stores/databases. |
| 1.8.0 | 2020-07 | ESM build support. |
| 1.10.0 | 2021-08 | `dropInstance` error handling; final release to date[^3]. |

## References

[^1]: Mozilla Hacks, "localForage: Offline Storage, Improved" — 2014.
    https://hacks.mozilla.org/2014/02/localforage-offline-storage-improved/
[^2]: Chrome for Developers, "Deprecating and removing Web SQL".
    https://developer.chrome.com/blog/deprecating-web-sql
[^3]: localForage GitHub releases (0.2.0 2014-03-20 … 1.10.0 2021-08-18).
    https://github.com/localForage/localForage/releases
[^4]: WebKit blog, "Full Third-Party Cookie Blocking and More" — 2020-03-24
    (7-day cap on script-writable storage).
    https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/

## Tags

javascript, browser-storage, indexeddb, localstorage, websql, key-value,
offline-first, pwa, client-side, library
