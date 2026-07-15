# redis/lettuce

> Netty-based Java Redis client whose single connection is shared across threads for non-blocking sync, async, and reactive access.

[GitHub repo](https://github.com/redis/lettuce) ·
[Official website](https://lettuce.io) ·
[License: MIT](https://github.com/redis/lettuce/blob/main/LICENSE)

## Overview

Lettuce is one of the two mainstream Java clients for Redis, the other being Jedis. It began as a fork of Will Glozer's original `wg/lettuce`[^1], was carried for years by Mark Paluch as `lettuce-io`, and now lives under the `redis/lettuce` organization as an officially adopted client. It is built on Netty and Project Reactor, and exposes the same command surface through three API flavors: a blocking synchronous facade, a `RedisFuture`-based async API, and a Reactor `Mono`/`Flux` reactive API[^2].

The defining design choice — and the source of most confusion — is that a Lettuce connection is thread-safe and meant to be shared. Where Jedis hands each thread a pooled connection, Lettuce multiplexes many threads' commands over one Netty channel. This is efficient and non-blocking right up until you issue a command that holds the connection: blocking pops (`BLPOP`, `BRPOP`, `XREAD BLOCK`) and transactions (`MULTI`/`EXEC`) serialize everything else on that channel, so those cases need a dedicated connection.

Lettuce is the default Redis driver behind Spring Data Redis and therefore Spring Boot's `spring-boot-starter-data-redis`[^3], which means a large share of Java services depend on it transitively without their authors ever choosing it directly.

## Getting Started

```xml
<dependency>
  <groupId>io.lettuce</groupId>
  <artifactId>lettuce-core</artifactId>
  <version>6.x.x</version>
</dependency>
```

```java
// Synchronous
RedisClient client = RedisClient.create("redis://localhost:6379");
StatefulRedisConnection<String, String> connection = client.connect();
RedisCommands<String, String> sync = connection.sync();
sync.set("key", "value");
String value = sync.get("key");   // blocks the calling thread on the future

connection.close();
client.shutdown();
```

```java
// Reactive — nothing runs until subscribe/block
RedisReactiveCommands<String, String> reactive = connection.reactive();
Mono<String> get = reactive.set("key", "value").then(reactive.get("key"));
get.block();
```

## Architecture / How It Works

A `RedisClient` owns a Netty event-loop group and connection resources; connecting returns a `StatefulRedisConnection` that wraps a single channel. The `.sync()`, `.async()`, and `.reactive()` methods are three views over the same underlying command dispatch — the sync facade is a thin blocking wrapper around the async futures, which are in turn completed by Netty's I/O threads. Because dispatch is non-blocking, one connection serves arbitrary concurrency without a pool.

Serialization is handled by `RedisCodec` implementations (`StringCodec`, `ByteArrayCodec`, and custom codecs for JSON/protobuf). Commands are generated from interfaces named after the lowercase Redis command; modifiers that change the result type are folded into the method name (`zrangebyscoreWithScores`).

For high availability Lettuce supports Redis Sentinel, Redis Cluster, and static Master/Replica topologies, each with its own connection type and options object (`ClusterClientOptions`, etc.). In cluster mode it maintains a client-side slot map and routes each key to the owning node. RESP3 (the Redis 6 protocol with push messages and richer types) is supported and negotiable since Lettuce 6.0[^4]. Transport can be plain NIO or a native transport — epoll, kqueue, or io_uring — selected automatically when the matching Netty native library is on the classpath.

The reactive layer is Project Reactor, not RxJava; command interfaces return `Mono` and `Flux`, and the usual reactive rule applies — no request is sent until something subscribes.

## Production Notes

**Cluster topology refresh is off by default.** This is the single most common Lettuce footgun. Unless you explicitly enable periodic and/or adaptive refresh via `ClusterTopologyRefreshOptions`, Lettuce will not pick up resharding, failover, or added nodes, and you will see stale-slot errors after a cluster change[^5]. Almost every production cluster deployment needs this turned on.

**Disconnect buffering can hide outages and grow memory.** By default commands issued while the connection is down are queued and auto-reconnect replays them. If Redis is unavailable for a while, this queue can grow unbounded and mask the failure instead of failing fast. Tune `ClientOptions` — `disconnectedBehavior`, `requestQueueSize`, and `rejectCommandsWhileDisconnected` — for the behavior you actually want.

**Don't block the shared connection.** Any blocking command or `MULTI`/`EXEC` transaction stalls all other traffic on that channel. Use a separate dedicated connection (or a connection pool via `commons-pool2` with `ConnectionPoolSupport`) for blocking pops and transactional work.

**Don't block Netty threads in reactive code.** Calling `.block()` or doing blocking I/O inside a reactive operator running on the event loop can deadlock or starve the I/O threads. Publish to a bounded scheduler for blocking work.

**Timeouts and pooling.** The default command timeout is 60 seconds — long enough to look like a hang under load; set it per `RedisURI` or `ClientOptions`. Lettuce needs no pool for normal non-blocking use, so pooling one connection per thread (Jedis-style) is an anti-pattern here.

## When to Use / When Not

**Use when:**
- You want async or reactive Redis access, or you're already on Spring Data Redis (Lettuce is its default).
- You want high concurrency over few connections without managing a pool.
- You need Cluster, Sentinel, or Master/Replica with SSL and native transports.

**Avoid when:**
- You want the simplest possible blocking client and are comfortable with a connection pool — Jedis has a shorter learning curve.
- You need Redis-backed distributed objects, locks, and data structures out of the box — that is Redisson's job, not a thin command client's.
- You cannot take a Netty and Reactor dependency footprint.

## Alternatives

- redis/jedis — simpler blocking client with pooled connections; use when you want straightforward synchronous access and don't need reactive or connection multiplexing.
- redisson/redisson — higher-level distributed objects, locks, and services; use when you want Redis as a distributed-primitives backend rather than a raw command client.
- spring-projects/spring-data-redis — template/repository abstraction that sits on top of Lettuce or Jedis; use when you're in Spring and want the abstraction instead of the driver directly.
- vert.x-redis-client — reactive Redis for the Vert.x stack; use when your app is already Vert.x rather than Reactor/Spring.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-05 | Repository created; fork of `wg/lettuce`[^1]. |
| 4.0 | 2016 | Netty 4 rebuild, async-first API. |
| 5.0 | 2017-11 | Reactive API on Project Reactor; groupId moved to `io.lettuce`[^2]. |
| 6.0 | 2020-10 | RESP3 protocol support, Redis 6 features, dynamic command interfaces[^4]. |
| 6.x | ongoing | Cluster/Sentinel refinements, io_uring transport, adopted under `redis/` org. |

## References

[^1]: Lettuce README, "License / Fork of https://github.com/wg/lettuce." https://github.com/redis/lettuce/blob/main/README.md
[^2]: Lettuce reference guide, API flavors (synchronous, asynchronous, reactive). https://redis.github.io/lettuce/
[^3]: Spring Data Redis reference — Lettuce is the default connection driver. https://docs.spring.io/spring-data/redis/reference/redis/drivers.html
[^4]: Lettuce 6.0 release notes — RESP3 support. https://github.com/redis/lettuce/releases/tag/6.0.0.RELEASE
[^5]: Lettuce reference guide, "Redis Cluster — Refreshing the cluster topology view." https://redis.github.io/lettuce/ha-sharding/#redis-cluster

## Tags

java, redis, redis-client, netty, reactive, project-reactor, async, redis-cluster, redis-sentinel, database-client, jvm
