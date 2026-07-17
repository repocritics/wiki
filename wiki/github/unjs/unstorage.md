# unjs/unstorage

> An async key-value storage API with a tiny core and dozens of pluggable drivers, unified behind one interface.

[GitHub repo](https://github.com/unjs/unstorage) ·
[Official website](https://unstorage.unjs.io) ·
[License: MIT](https://github.com/unjs/unstorage/blob/main/LICENSE)

## Overview

Unstorage is a storage abstraction from the UnJS collective (the tooling group behind Nitro, Nuxt 3, ofetch, and h3)[^1]. It presents a single async key-value API — `getItem`, `setItem`, `removeItem`, `getKeys`, and a handful of helpers — and routes every call to a pluggable driver. The core ships as a small, tree-shakable module; each driver (memory, filesystem, Redis, Cloudflare KV, S3, HTTP, and dozens more) lives behind a subpath import so you only bundle what you mount.

Its defining trait is Unix-style **mounting**: you can attach different drivers at different key prefixes on one storage instance, so `cache:*` hits Redis while `assets:*` hits the filesystem, all through the same handle. This is the mechanism Nitro uses to give Nuxt apps a uniform `useStorage()` regardless of deploy target[^2]. As of 2026 the repo has ~2,600 stars and is pushed to continuously; it is effectively load-bearing infrastructure for the Nuxt/Nitro ecosystem rather than a standalone side project.

The central tradeoff is the usual abstraction tax: unstorage exposes the *intersection* of what its drivers can do. You get portability and driver-swapping for free, but you do not get Redis pipelines, atomic increments, native TTL guarantees, or transactions through the common API. It is a KV facade, not a database client.

## Getting Started

```sh
npm install unstorage
# pnpm add unstorage / yarn add unstorage
```

```js
import { createStorage } from "unstorage";
import fsDriver from "unstorage/drivers/fs";
import redisDriver from "unstorage/drivers/redis";

const storage = createStorage(); // defaults to in-memory

// Unix-style mounts: prefix -> driver
storage.mount("cache", redisDriver({ base: "app:" }));
storage.mount("data", fsDriver({ base: "./.data" }));

await storage.setItem("cache:user:1", { name: "Ada" }); // JSON auto-serialized
const user = await storage.getItem("cache:user:1");
const keys = await storage.getKeys("cache");          // list under a prefix
await storage.setItemRaw("data:blob", new Uint8Array([1, 2, 3])); // binary
```

Keys are path-like; `:` and `/` are treated as the same separator and normalized (`foo/bar` and `foo:bar` resolve to the same key). The unmounted root uses the default in-memory driver.

## Architecture / How It Works

The core is a thin dispatcher over a **mount tree**. Every operation resolves the longest matching mount prefix for a key, strips it, and forwards the remainder to that driver. With no mounts, everything lands on the default memory driver.

A **driver** is a plain object implementing a small contract — `hasItem`, `getItem`, `setItem`, `removeItem`, `getKeys`, `clear`, `dispose`, and optionally `getMeta`, `watch`, and the raw/`*Raw` variants for binary. Because the surface is small, community drivers are easy to write; the tradeoff is that optional methods are genuinely optional, so capability varies per backend.

Values pass through auto-serialization: objects are `JSON.stringify`'d on write and parsed on read. The `getItemRaw`/`setItemRaw` pair bypass JSON to move binary/`Uint8Array` payloads, but not every driver implements raw storage faithfully — some fall back to base64 or string coercion.

Other core pieces:

- **`prefixStorage(storage, "base")`** — returns a namespaced view without a new mount, useful for scoping without reconfiguring the tree.
- **Snapshots** — `snapshot()`/`restoreSnapshot()` serialize an entire (sub)tree to a plain object for seeding, testing, or migration[^3].
- **Watching** — `storage.watch(cb)` emits `update`/`remove` events, but only drivers that implement `watch` fire natively; others are effectively no-ops.
- **Metadata** — `getMeta`/`setMeta` expose `mtime`, `size`, and custom fields where the driver supports it.
- **HTTP server** — `createStorageServer` exposes a storage instance over HTTP, and the `http` driver consumes that same protocol, so one process's storage can back another over the wire[^4].

Drivers are imported from `unstorage/drivers/*` rather than the package root, which keeps the core tiny and lets bundlers drop unused backends.

## Production Notes

- **`getKeys` is not free.** On the filesystem driver it walks directories; on Redis it uses key scanning. Calling `getKeys()` on a large unbounded namespace can be O(everything). Scope it with a prefix and treat it as an admin/maintenance operation, not a hot path.
- **TTL is driver-dependent.** Some drivers accept a `ttl` option and translate it to native expiry (Redis), others silently ignore it. Do not assume a `setItem` TTL is enforced unless you have confirmed the specific driver honors it. The common API gives no cross-driver expiry guarantee.
- **No atomicity or transactions.** There are no atomic increments, compare-and-set, or multi-key transactions. If you need those, talk to Redis/your database directly for that path and reserve unstorage for the portable cache/blob use cases.
- **Key normalization surprises.** Because `/` and `:` collapse to one separator and keys are normalized, two paths you think are distinct can alias. Pick one separator convention and keep keys opaque.
- **Raw/binary fidelity varies.** Verify `getItemRaw` round-trips bytes exactly on your chosen driver before relying on it for images/buffers; JSON-oriented backends may not preserve binary cleanly.
- **`watch` portability.** Reactive invalidation works locally (memory/fs) but should not be assumed on remote KV drivers. Design cache invalidation around explicit writes, not watchers, if you need it everywhere.
- **Mount ordering / disposal.** `dispose()` should be called to close driver connections (Redis clients, file handles) on shutdown; long-lived serverless instances that skip it can leak connections.

## When to Use / When Not

**Use when:**
- You want one storage API that swaps between memory, filesystem, and a cloud KV per environment without code changes (the Nitro/Nuxt deploy-target story).
- You need a small, tree-shakable cache/blob layer with many ready-made drivers.
- You want mountable namespaces (different backends under different prefixes) behind a single handle.
- You're already in the UnJS/Nitro ecosystem and want the idiomatic option.

**Avoid when:**
- You need database semantics: transactions, atomic counters, secondary indexes, or queries. Use the real client.
- You depend on guaranteed TTL/expiry or native pub/sub — reach for Redis directly.
- Binary fidelity and streaming large blobs are central; an object store SDK (S3) gives you multipart/streaming the facade won't.
- You want the richest features of one specific backend — the common API is a lowest-common-denominator by design.

## Alternatives

- jaredwray/keyv — the other established multi-backend KV facade; Map-like API, mature adapter set. Use when you want a simpler flat interface without mounting.
- redis/ioredis — use directly when you need Redis-specific features (pipelines, Lua, pub/sub, atomic ops) rather than a portable subset.
- isaacs/node-lru-cache — use for a purely in-process bounded cache with no persistence or driver abstraction.
- vercel/storage (`@vercel/kv`, `@vercel/blob`) — use when you're all-in on Vercel and want their managed KV/blob directly instead of a portable layer.
- cacheable — use when caching (TTL, stale-while-revalidate) is the primary goal rather than general KV storage.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-03 | Repo created; extracted as a UnJS storage primitive[^1]. |
| 1.0 | 2022 | First stable major; mount API, JSON serialization, snapshots. |
| 1.x | 2023–2024 | Broad driver expansion (Cloudflare, S3, Vercel, Netlify, etc.), raw/binary support, metadata. |
| 1.x | 2026-07 | Actively maintained; continuous pushes, part of the Nitro/Nuxt stack. |

## References

[^1]: unstorage documentation and UnJS project. https://unstorage.unjs.io
[^2]: Nitro storage layer (built on unstorage). https://nitro.build/guide/storage
[^3]: Snapshots & utils. https://unstorage.unjs.io/getting-started/utils#snapshots
[^4]: HTTP server & driver. https://unstorage.unjs.io/guide/http-server

## Tags

typescript, key-value-store, storage, cache, drivers, unjs, nitro, javascript, nodejs, edge, abstraction-layer
