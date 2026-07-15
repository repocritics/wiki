# StackExchange/StackExchange.Redis

> A high-throughput .NET Redis client built around a single multiplexed connection rather than a connection pool.

[GitHub repo](https://github.com/StackExchange/StackExchange.Redis) ·
[Official website](https://stackexchange.github.io/StackExchange.Redis/) ·
[License: MIT](https://github.com/StackExchange/StackExchange.Redis/blob/main/LICENSE)

## Overview

StackExchange.Redis is the Redis client extracted from Stack Overflow's own infrastructure and open-sourced in 2014[^1]. It talks the RESP protocol, so it also targets Redis-compatible servers: Valkey, Garnet, Azure Managed Redis, and AWS ElastiCache among them[^2]. It is the default Redis client in the .NET ecosystem and the transport under `Microsoft.Extensions.Caching.StackExchangeRedis` and Redis Inc.'s own `NRedisStack`.

The defining design decision is **multiplexing**. Instead of a pool of connections handed out per operation, the library keeps one (or a few) long-lived TCP connections shared by the entire process and pipelines all commands over them. A `ConnectionMultiplexer` is meant to be created once and reused for the lifetime of the application. This buys very high throughput and low connection count against the server, at the cost of a programming model that surprises people coming from pool-based clients like ServiceStack.Redis or Java's Jedis.

The central tension follows directly from that choice: because the connection is shared, any command that would block the socket for one caller would block it for everyone. StackExchange.Redis therefore refuses to support blocking operations (`BLPOP`, `BRPOP`, blocking `WAIT`), and its most common production failure — the "timeout awaiting response" exception — is almost always a symptom of contention on the shared pipe or of .NET thread-pool starvation rather than of Redis itself[^3].

## Getting Started

```bash
dotnet add package StackExchange.Redis
```

```csharp
using StackExchange.Redis;

// Create ONCE per process and keep it. Never per-request.
ConnectionMultiplexer redis =
    await ConnectionMultiplexer.ConnectAsync("localhost:6379");

IDatabase db = redis.GetDatabase();          // cheap; grab freely

await db.StringSetAsync("user:1:name", "Tom");
string? name = await db.StringGetAsync("user:1:name");

// Server-level commands (KEYS, FLUSHDB) are NOT on IDatabase:
IServer server = redis.GetServer("localhost", 6379);
foreach (RedisKey key in server.Keys(pattern: "user:*"))
    Console.WriteLine(key);
```

In ASP.NET Core the `ConnectionMultiplexer` is registered as a singleton (or the `AddStackExchangeRedisCache` helper is used) so the single-instance rule is enforced by DI.

## Architecture / How It Works

**`ConnectionMultiplexer`** is the root object and the expensive one. It owns the physical connections, tracks server topology (standalone, Sentinel, or Cluster), handles reconnection, and hides the difference between a single node and a sharded cluster. `GetDatabase()` returns a lightweight `IDatabase` handle — it is a cheap struct-backed facade, not a new connection, so callers create them freely.

**Multiplexing and pipelining.** All commands from all threads are funneled onto the shared connection and written back-to-back without waiting for each reply (pipelining). Replies are matched to their pending tasks as they arrive. This is why one connection can sustain enormous command rates, and why ordering across separate `await` calls is not guaranteed unless you use a transaction or a Lua script.

**Value types avoid allocation.** `RedisKey` and `RedisValue` are `struct` types with implicit conversions from `string`, `byte[]`, `int`, etc., so the common path does not allocate wrapper objects per command.

**The 2.0 rewrite** moved the internals onto `System.IO.Pipelines` (via `Pipelines.Sockets.Unofficial`), cutting allocations and buffer copies on the socket path[^4]. This is a significant internal boundary: behavior, threading, and timeout semantics differ between the 1.x and 2.x/3.x lines.

**Pub/Sub** uses a dedicated connection inside the multiplexer via `GetSubscriber()`, so a long-lived subscription does not stall normal command traffic on the primary pipe.

**Transactions** (`CreateTransaction`) map to `MULTI`/`EXEC` with preconditions expressed as `Condition` objects rather than interactive `WATCH`. You cannot read a value mid-transaction and branch on it — every queued result only materializes after `EXEC`. For read-modify-write atomicity, Lua via `ScriptEvaluate` is the idiomatic escape hatch.

## Production Notes

**Timeout exceptions are the signature failure.** `RedisTimeoutException` / "Timeout awaiting response" is rarely a slow Redis. The usual causes are (1) .NET thread-pool starvation — synchronous-over-async code exhausts worker threads so continuations can't run; raising `ThreadPool.SetMinThreads` is the standard mitigation; (2) large payloads head-of-line-blocking the shared pipe, since a multi-megabyte value serializes ahead of everyone else's small commands; (3) genuinely slow `O(N)` commands like `KEYS` or big `LRANGE`[^3]. The exception message includes queue depth counters (`qs`, `in`, `busyworkers`) specifically to distinguish these cases.

**One multiplexer, reused.** Creating a `ConnectionMultiplexer` per request is the most common misuse and exhausts sockets and CPU. It is thread-safe and designed to be a singleton.

**No blocking commands.** `BLPOP`/`BRPOP`/blocking `WAIT` are unavailable by design. Use pub/sub, polling, or streams instead.

**Reconnection on managed Redis.** On Azure and other cloud Redis, connections drop during failovers and patching. Connect with `abortConnect=false` so startup does not throw when a node is briefly unavailable, and set `ConnectRetry` / `ConnectTimeout` appropriately. Older versions recovered poorly from certain socket states, which led to Microsoft's widely copied "ForceReconnect" helper pattern; auto-reconnect improved in later 2.x releases but the connection-string hardening is still recommended[^5].

**Server vs database commands.** Administrative and keyspace-wide commands (`KEYS`, `SCAN`, `FLUSHDB`, `CONFIG`, `INFO`) live on `IServer` obtained from `GetServer()`, not `IDatabase`, because they target a specific node — an important distinction under Cluster.

**`CommandFlags.FireAndForget`** discards the reply for write-heavy paths where you do not need confirmation, trading durability of acknowledgement for throughput.

## When to Use / When Not

**Use when:**
- You are on .NET and want the de facto, battle-tested Redis client (it runs Stack Overflow).
- You want maximum throughput with minimal connections via multiplexing and pipelining.
- You target Redis or a RESP-compatible server (Valkey, Garnet, ElastiCache, Azure).
- You want tight integration with ASP.NET Core distributed caching and data-protection key rings.

**Avoid when:**
- You need blocking pop semantics or a classic per-operation connection-pool model — a pooled client fits better.
- You want first-class support for Redis Stack modules (JSON, Search, Bloom) out of the box — use the module-aware client instead.
- Your workload is dominated by large values, where head-of-line blocking on the shared pipe hurts tail latency and a pooled/isolated-connection design may serve better.

## Alternatives

- ServiceStack/ServiceStack.Redis — pooled client that does support blocking operations; use when you want a connection-pool model, but note its free-tier quota then commercial licensing.
- redis/NRedisStack — Redis Inc.'s official .NET client; it builds on top of StackExchange.Redis and adds Redis Stack module APIs (JSON, Search, TimeSeries). Use when you need those modules.
- 2881099/FreeRedis — MIT-licensed .NET client with connection pooling and support for commands SE.Redis omits (including blocking pops). Use when you want pooling plus a broader command surface.
- redis/go-redis / redis/lettuce — use when the service is not on .NET at all and you want the mainstream client for Go or the JVM.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014 | Open-sourced from Stack Overflow's internal client[^1]. |
| 2.0 | 2018 | Rewrite onto `System.IO.Pipelines`; lower allocations, changed timeout/threading behavior[^4]. |
| 2.1 | 2020 | Broad 2.x line; expanded command coverage and cluster/Sentinel work. |
| 2.13 | 2026-05 | Last 2.x maintenance line before 3.0[^2]. |
| 3.0.0 | 2026-06-12 | Current major line[^2]. |
| 3.0.17 | 2026-07-10 | Latest release at time of writing[^2]. |

## References

[^1]: Marc Gravell, "StackExchange.Redis" announcement — Stack Exchange, 2014. https://blog.marcgravell.com/2014/03/so-i-went-and-wrote-another-redis-client.html
[^2]: StackExchange.Redis documentation and release notes. https://stackexchange.github.io/StackExchange.Redis/
[^3]: StackExchange.Redis docs, "Timeouts". https://stackexchange.github.io/StackExchange.Redis/Timeouts
[^4]: StackExchange.Redis docs, "PipeLines and NetworkStreams" / 2.0 pipelines rewrite. https://stackexchange.github.io/StackExchange.Redis/
[^5]: Microsoft Azure Cache for Redis — best practices for connection resilience (ForceReconnect pattern). https://learn.microsoft.com/azure/azure-cache-for-redis/cache-best-practices-connection

## Tags

csharp, dotnet, redis, redis-client, valkey, cache, database, multiplexing, pub-sub, networking
