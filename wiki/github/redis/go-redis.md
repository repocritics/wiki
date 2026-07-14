# redis/go-redis

> The official Redis client for Go — a full-command, connection-pooling driver with a command-object API and built-in cluster, sentinel, and RESP3 support.

[GitHub repo](https://github.com/redis/go-redis) ·
[Official website](https://redis.io) ·
[License: BSD-2-Clause](https://github.com/redis/go-redis/blob/master/LICENSE)

## Overview

go-redis is the Go client library that Redis Inc. designates as official. It began life as `go-redis/redis`, written and maintained for years by Vladimir Mihailenco (the author of the Uptrace observability tooling), and was later adopted into the `redis` GitHub organization as the canonical Go driver[^1]. The current major version is v9, imported as `github.com/redis/go-redis/v9`.

The library's defining design choice is its **command-object API**: every command returns a typed `*Cmd` value (e.g. `*StringCmd`, `*IntCmd`, `*SliceCmd`) rather than a raw reply. You call `.Result()`, `.Val()`, or `.Err()` on it to extract a typed value and error. This gives you Go-native return types and IDE autocompletion for the several hundred Redis commands, at the cost of a large generated-feeling surface area and one wrapper type per return shape. The alternative philosophy — a single generic `Do()` that returns an untyped reply — is available via `rdb.Do(ctx, ...)` but is not the primary interface.

The other load-bearing convention is that **every command takes a `context.Context` as its first argument**. This was introduced in v8 and is not optional; it threads cancellation, deadlines, and tracing spans through every call. Missing keys are not errors in the Go sense — they surface as the sentinel `redis.Nil`, which callers must test for explicitly (`if err == redis.Nil`).

## Getting Started

```shell
go get github.com/redis/go-redis/v9
```

```go
package main

import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // no password
        DB:       0,
    })
    defer rdb.Close()

    if err := rdb.Set(ctx, "key", "value", 0).Err(); err != nil {
        panic(err)
    }

    val, err := rdb.Get(ctx, "key").Result()
    if err == redis.Nil {
        fmt.Println("key does not exist")
    } else if err != nil {
        panic(err)
    } else {
        fmt.Println("key:", val)
    }
}
```

`redis.ParseURL("redis://user:pass@host:6379/0?protocol=3")` builds `*Options` from a connection string, including the `protocol` query param to select RESP2 vs RESP3.

## Architecture / How It Works

There is not one client type but a family, each wrapping the same command set over a different topology:

- **`Client`** — a single-node connection with an internal connection pool.
- **`ClusterClient`** — Redis Cluster. Maintains a slot→node map, routes each key to its owning shard, and transparently follows `MOVED`/`ASK` redirects when slots migrate.
- **`FailoverClient`** — Sentinel-managed high availability; discovers the current master through Sentinel and reconnects on failover.
- **`Ring`** — client-side sharding across several independent Redis instances using consistent hashing, with no server-side cluster. Useful when you want horizontal partitioning without running Cluster.

All of them share the **connection pool**, which is automatic and always on. Pool sizing (`PoolSize`, `MinIdleConns`, `MaxIdleConns`, `PoolTimeout`, `ConnMaxLifetime`, `ConnMaxIdleTime`) is the main tuning surface for throughput and connection churn.

**Pipelines and transactions** are first-class: `Pipeline()` / `Pipelined()` batch commands into one round trip, and `TxPipeline()` wraps them in `MULTI`/`EXEC`. Optimistic locking uses `Watch()` with a `WATCH`/`MULTI`/`EXEC` closure that retries on `TxFailedErr`.

**Hooks** let you wrap three interception points — `DialHook`, `ProcessHook`, and `ProcessPipelineHook` — around connection dialing and command execution. This is how cross-cutting concerns are layered on; OpenTelemetry tracing and metrics are shipped as a separate `extra/redisotel` module built entirely on hooks, rather than baked into the core.

**RESP3** support arrived with v9. The client speaks both RESP2 and RESP3 (selected by `Protocol: 3`), and RESP3's typed replies (maps, doubles, push messages) are what make attributes like command result metadata and client-side push available.

## Production Notes

**Versioned import paths are a real migration cost.** Following Go module semantics, each major (v6, v7, v8, v9) lives at a distinct import path. Upgrading is not a `go get -u`; it is a source change across every import and often an API change. The v7→v8 jump added `context.Context` to every call signature — a mechanical but repo-wide edit. The v8→v9 jump moved the canonical import to `github.com/redis/go-redis/v9` and changed RESP3/Redis 7 behaviors. Plan majors as deliberate migrations, not routine bumps.

**`redis.Nil` is the classic footgun.** A cache miss returns `redis.Nil`, not an empty string with `nil` error. Code that only checks `err != nil` will treat every miss as a hard failure; code that ignores the error entirely will read a zero value silently. Every `Get`-style call needs the three-way branch.

**Context cancellation kills in-flight commands.** Because the context is threaded to the socket, a cancelled or timed-out context aborts the command and can return the connection to the pool in an ambiguous state. Set per-call timeouts deliberately; a too-aggressive `context.WithTimeout` under load produces spurious `context deadline exceeded` errors that look like Redis problems but are client-side.

**Pool exhaustion presents as `PoolTimeout`.** If `PoolSize` is too small for concurrency, goroutines block waiting for a connection and eventually error with a pool timeout, not a Redis error. The default pool size scales with `GOMAXPROCS`; high-concurrency services usually need to raise it explicitly.

**Cluster and pipelines interact awkwardly.** A pipeline against `ClusterClient` can span keys on different shards; the client splits and reissues per node, so atomicity guarantees you'd expect from a single-node `MULTI` do not hold across slots. Keep transactional pipelines to a single hash slot (use hash tags `{...}`).

**Typed error helpers** (`redis.IsLoadingError`, `IsReadOnlyError`, `IsClusterDownError`, `IsMovedError`, `IsAskError`, `IsTryAgainError`) let you build retry logic without string-matching server error prefixes. When wrapping errors in hooks, call `cmd.SetErr()` and preserve `Unwrap()` so these detectors keep working through your wrappers.

## When to Use / When Not

**Use when:**
- You want the vendor-endorsed Go client with the widest command coverage and active maintenance.
- You need Cluster, Sentinel, or Ring topologies behind one consistent API.
- You want typed return values and integrated OpenTelemetry rather than hand-decoding replies.

**Avoid when:**
- You need maximum throughput and want automatic pipelining plus client-side caching — a client built around those goals will outperform go-redis under high concurrency.
- You want a minimal, low-magic driver where you decode replies yourself — a thinner client has less surface to learn.
- You cannot absorb the import-path churn of a major upgrade in a large codebase.

## Alternatives

- redis/rueidis — high-performance Go client with automatic pipelining and RESP3 client-side caching; use when raw throughput and opportunistic caching matter more than API familiarity.
- gomodule/redigo — older, lower-level `Do()`-style client with manual reply decoding; use when you want a thin driver and don't need typed command methods.
- mediocregopher/radix — pluggable, connection-pool-focused client; use when you want fine control over pooling and pipelining internals.
- bsm/redislock — not a client but a distributed-lock library built on go-redis; use alongside go-redis for Redlock-style locking.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-07 | Repository created as go-redis/redis[^1]. |
| v6 | 2016 | Long-lived major; pre-context API. |
| v7 | 2019 | Cluster and API refinements. |
| v8 | 2020 | `context.Context` required on every command[^2]. |
| v9 | 2023 | Moved to `redis/go-redis/v9` import path; RESP3 support, Redis 7 features[^3]. |

## References

[^1]: go-redis repository, `redis/go-redis` (formerly `go-redis/redis`). https://github.com/redis/go-redis
[^2]: go-redis v8 release notes — context added to command signatures. https://github.com/redis/go-redis/releases
[^3]: Redis Go client documentation. https://redis.io/docs/latest/develop/clients/go/

## Tags

go, redis, redis-client, database, cache, key-value-store, connection-pooling, resp3, redis-cluster, driver
