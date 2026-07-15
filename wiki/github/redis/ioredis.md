# redis/ioredis

> A full-featured Redis client for Node.js with first-class Cluster and Sentinel support — now in best-effort maintenance mode.

[GitHub repo](https://github.com/redis/ioredis) ·
[API docs](https://redis.github.io/ioredis/) ·
[License: MIT](https://github.com/redis/ioredis/blob/main/LICENSE)

## Overview

ioredis is a Redis client for Node.js, originally written by Zihua Li (`luin`) and later adopted into the official `redis` GitHub organization[^1]. For most of the late 2010s it was the de facto production Redis client for Node, largely because it shipped Cluster and Sentinel support before the older `node-redis` did, and because its API passes command arguments straight through to Redis rather than wrapping each command in a bespoke signature. It runs at large scale — the README cites Alibaba as a user — and is written 100% in TypeScript with bundled type declarations.

The defining fact about ioredis in 2026 is its status. The maintainers have declared it a **stable project in best-effort maintenance mode**: relevant issues and beneficial contributions are still reviewed and merged, but new development has moved to `node-redis`, which Redis now recommends for new projects[^2]. `node-redis` is where hash-field expiration, Redis 8 / Redis Stack modules (search, JSON, time-series, probabilistic structures), and future commands land first. ioredis remains compatible with Redis 7.x and supports Redis >= 2.6.12, so existing deployments are not stranded — but the trajectory is frozen. New greenfield projects choosing ioredis in 2026 are choosing a mature, well-understood client that will not gain new server-feature coverage.

The library's enduring appeal is its ergonomics under load: transparent EVALSHA caching for Lua scripts, autopipelining, key prefixing, offline command queueing, and a Cluster/Sentinel layer that most teams never had to think hard about. Its enduring hazards are the ones that come from a "pass everything through" design — argument and reply shapes are Redis's, not the library's, so correctness depends on the caller knowing the protocol.

## Getting Started

```shell
npm install ioredis
```

```javascript
const Redis = require("ioredis");
// TypeScript: import { Redis } from "ioredis";

const redis = new Redis(); // defaults to 127.0.0.1:6379

await redis.set("mykey", "value", "EX", 10); // SET mykey value EX 10
const value = await redis.get("mykey");      // "value"

// Arguments are passed directly to Redis — the library does not
// reshape them. Replies come back in Redis's own layout:
await redis.zadd("board", 1, "one", 2, "two", 3, "three");
const rows = await redis.zrange("board", 0, -1, "WITHSCORES");
// ["one", "1", "two", "2", "three", "3"]
```

Every bulk-string command has a `Buffer` variant (`getBuffer`, `hgetallBuffer`) that returns raw buffers instead of UTF-8 strings for binary data.

## Architecture / How It Works

**Command dispatch is dynamic.** ioredis does not hand-write a method per Redis command. It generates methods from Redis's command table, joins arguments into a command string, and lets the server validate. The last argument, if a function, is treated as a Node-style callback; otherwise the call returns a Promise. This is why "technically ioredis supports all Redis commands" — including ones newer than the library, as long as the server understands them. The cost is that the client cannot type-check or reshape arguments it doesn't know about.

**Connection model.** Each `Redis` instance owns one TCP connection and connects eagerly on construction. A single connection cannot be both a publisher and a subscriber: calling `subscribe()`/`psubscribe()` puts the connection into Redis's subscriber mode, where only subscription-management commands are legal until the subscription set empties. Pub/sub therefore requires a dedicated second instance.

**Pipelining and transactions.** `redis.pipeline()` queues commands in memory and flushes them in one round trip; the README cites 50%–300% throughput gains for batches. `multi()` is pipeline-backed by default (a `MULTI`/`EXEC` transaction assembled client-side), with `{ pipeline: false }` to send each command immediately. **Autopipelining** (`enableAutoPipelining`) transparently batches commands issued within the same event-loop tick, giving pipeline throughput without manual batching.

**Lua scripting.** `defineCommand()` registers a named script; ioredis caches the SHA and automatically chooses `EVALSHA` vs `EVAL`, transparently falling back on `NOSCRIPT`. Defined commands behave like native ones, including in pipelines and with `Buffer` variants.

**Cluster and Sentinel.** `new Redis.Cluster([...])` maintains slot-to-node mapping, follows `MOVED`/`ASK` redirections, and refreshes topology on failover. Sentinel mode discovers the current master through sentinel nodes and reconnects on promotion. NAT mapping exists specifically so Cluster works when nodes advertise internal addresses behind a gateway.

## Production Notes

**Maintenance status is the first operational fact.** Bugs get best-effort fixes; new Redis 8 / Redis Stack server features (JSON, search, time-series, probabilistic types, hash-field TTL) are not being added. If a roadmap depends on those, plan for `node-redis` rather than expecting ioredis to catch up[^2].

**The offline queue is a silent footgun.** By default, commands issued before the connection is ready (or while it is down) are queued in memory and replayed on reconnect. This masks connectivity problems and, under a long outage with high write volume, grows unbounded memory. For request-path code that should fail fast, set `enableOfflineQueue: false` and/or `maxRetriesPerRequest` so calls reject instead of hanging.

**`maxRetriesPerRequest` defaults to 20.** A command that keeps failing retries up to 20 times before rejecting, which can extend tail latency during a Redis blip far beyond a caller's own timeout. Tune it (or set it to `null` to retry forever, or a low integer to fail fast) to match the surrounding request budget.

**Reply shapes are Redis's, not friendly objects.** `hgetall` returns a plain object, but list/zset/stream replies come back as flat or nested arrays exactly as RESP delivers them — e.g. `WITHSCORES` interleaves member and score as adjacent array elements, and `xread` returns `[key, [[id, [field, value, ...]], ...]]`. Parsing is the caller's job. RESP3 support and reply-typing are areas where `node-redis` has moved further.

**One connection per role, and pool accordingly.** Because a subscriber connection is monopolized, and because a single connection serializes commands, high-concurrency services typically run a small set of instances (a publish connection, a subscribe connection, and one or a few for regular commands) rather than one shared client. There is no built-in generic connection pool; teams either share a single pipelined instance or wrap several.

**v4 → v5 upgrade.** v5 dropped Node < 12 support and requires Node >= 12 (v4 supported Node >= 8). The default `import Redis from "ioredis"` still works but is slated for deprecation in favor of the named `{ Redis }` export. Consult the maintainers' v4→v5 upgrade guide for the breaking changes before bumping.

## When to Use / When Not

**Use when:**
- You run Redis Cluster or Sentinel and want a client that has handled failover and redirection in production for years.
- You have an existing ioredis codebase — there is no forced-migration pressure; it stays compatible with Redis 7.x.
- You want autopipelining, transparent Lua SHA caching, and pass-through command support without a per-command wrapper API.

**Avoid when:**
- You are starting a new project and expect to use Redis 8 / Redis Stack modules (JSON, search, time-series) — use node-redis, the recommended client for new work[^2].
- You want RESP3, typed replies, or the newest server commands supported day-one.
- You need a first-class connection pool and structured reply parsing out of the box rather than array-shaped RESP.

## Alternatives

- redis/node-redis — the officially recommended client for new projects; actively developed, RESP3, Redis 8 / Stack module coverage. Use instead when starting fresh or needing new server features.
- Uzlopak/redis-client (or the RESP-focused clients) — use when you want a leaner, lower-level RESP client without ioredis's feature surface.
- upstash/redis — use when targeting serverless/edge over HTTP rather than a persistent TCP connection.
- prisma/prisma or an ORM cache layer — use when you want caching abstracted rather than raw Redis command access.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-04 | First release by Zihua Li (`luin`); Cluster/Sentinel early. |
| 4.x | 2019 | Node >= 8; long-lived production line. |
| 5.x | 2022 | Node >= 12; named `{ Redis }` export; current line[^1]. |
| — | ~2022+ | Repo moved under the `redis` org; node-redis becomes recommended client[^2]. |

## References

[^1]: ioredis repository and README, `redis` GitHub organization. https://github.com/redis/ioredis
[^2]: ioredis README maintenance notice recommending node-redis for new projects. https://github.com/redis/node-redis

## Tags

nodejs, typescript, redis, redis-client, database, cache, redis-cluster, redis-sentinel, pub-sub, maintenance-mode
