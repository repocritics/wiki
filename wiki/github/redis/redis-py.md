# redis/redis-py

> The official Python client for Redis — a thin, near-1:1 mapping of the Redis command set onto Python methods, with connection pooling, cluster, and asyncio support built in.

[GitHub repo](https://github.com/redis/redis-py) ·
[PyPI: redis](https://pypi.org/project/redis/) ·
[Docs](https://redis.readthedocs.io/en/stable/) ·
[License: MIT](https://github.com/redis/redis-py/blob/master/LICENSE)

## Overview

redis-py is the reference Python client for the Redis key-value store. It was originally written by Andy McCurdy and later adopted and maintained by Redis Inc as the official client[^1]. The design philosophy is deliberately thin: rather than inventing an abstraction layer, it exposes Redis commands under their raw names (`r.set`, `r.hgetall`, `r.zadd`), so anyone who knows the Redis command reference already knows the API. There is no ORM, no query builder, no schema — that surface is delegated to sibling projects like redis-om-python.

The defining tension is between "thin transport client" and "batteries-included toolkit." Over time the single `redis` package has absorbed a lot: a synchronous client, a full asyncio client (`redis.asyncio`), cluster support (`RedisCluster`), Sentinel, connection pooling, PubSub, pipelines/transactions, and helpers for the Redis Stack modules (search, JSON, time-series, bloom). This breadth is convenient but means the client is large, carries two parallel code paths (sync and async) that occasionally drift in feature parity, and encodes a fair amount of Redis-server-version-specific behavior.

The second tension is Python's bytes-vs-str boundary. By default every response comes back as `bytes`; callers who want `str` must set `decode_responses=True` at the connection level. This choice is correct (Redis stores bytes) but is the single most common source of confusion for new users, and mixing decoded and non-decoded clients against the same keys produces subtle bugs.

## Getting Started

```bash
pip install redis
# optional C parser for faster response decoding:
pip install "redis[hiredis]"
```

```python
import redis

# decode_responses=True returns str instead of bytes
r = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)

r.set("foo", "bar")
print(r.get("foo"))          # "bar"

# pipelines batch commands into one round trip
with r.pipeline() as pipe:
    pipe.set("a", 1).incr("a").get("a")
    print(pipe.execute())    # [True, 2, "2"]
```

Async is a drop-in parallel API under `redis.asyncio`:

```python
import redis.asyncio as redis

async def main():
    r = redis.Redis(decode_responses=True)
    await r.set("foo", "bar")
    print(await r.get("foo"))
    await r.aclose()
```

## Architecture / How It Works

The stack is layered: a **Client** (`Redis`, `RedisCluster`, `Sentinel`) holds a **ConnectionPool**, which manages **Connection** objects, each of which owns a socket and a **parser**.

- **Command dispatch.** Client methods are mostly thin wrappers that assemble a command as a list of arguments and call `execute_command`. The pool checks out a connection, the command is written using the RESP protocol, and a response callback maps the raw reply into a Python type (e.g. `HGETALL` → `dict`, `SMEMBERS` → `set`). This callback table is why responses are typed rather than raw arrays.
- **Parsers.** Two exist: a pure-Python parser and `hiredis`, a C extension. If `hiredis` is installed it is used automatically; it is meaningfully faster on large replies and requires no code change.
- **Protocol.** RESP2 was the wire format for years. RESP3 support landed in 5.0 (`protocol=3`), enabling out-of-band push messages, native map/set/double types, and client-side caching invalidation. Newer versions default to RESP3 on the wire while preserving RESP2-shaped Python responses for compatibility[^2].
- **Cluster.** `RedisCluster` discovers the slot topology via `CLUSTER SLOTS`, hashes keys to slots (CRC16), and routes each command to the owning node, transparently following `MOVED`/`ASK` redirects. It maintains a pool per node.
- **Async.** `redis.asyncio` mirrors the sync API on top of asyncio transports. It began as the separate `aioredis` project, which was merged into redis-py in the 4.x line and is now the maintained async path[^3].

Pipelines wrap commands in `MULTI`/`EXEC` by default (`transaction=True`); PubSub is a separate object because a subscribed connection cannot issue normal commands.

## Production Notes

**Connection pools are per-client and not fork-safe.** Each `Redis()` instance gets its own pool unless you pass a shared `connection_pool`. After `os.fork()` (Gunicorn/uWSGI/Celery prefork), child processes inherit live sockets from the parent's pool — reusing them causes cross-talk and protocol desync. Create clients *after* fork (in a worker post-fork hook), or reset the pool in the child.

**Default pool raises instead of blocking.** The standard `ConnectionPool` raises `ConnectionError: Too many connections` when `max_connections` is exhausted. If you want callers to wait for a free connection instead, use `BlockingConnectionPool`. Many "random ConnectionError under load" incidents trace to this default.

**Thread safety is command-level only.** The `Redis` client is safe to share across threads for individual commands (each checks out its own connection). `Pipeline` and `PubSub` objects are **not** thread-safe and must not be shared across threads.

**Set timeouts explicitly.** By default there is no socket timeout, so a command against a wedged server or a silently dropped TCP connection can hang indefinitely. Set `socket_timeout`, `socket_connect_timeout`, and `health_check_interval` (which pings idle connections) for anything production-facing. Pair with `retry` / `retry_on_error` for transient failures.

**Cluster limits.** In `RedisCluster`, multi-key commands and `MULTI`/`EXEC` transactions only work when all keys hash to the same slot — use hash tags (`{user:1}:profile`) to co-locate. Cluster pipelines are grouped per node and are *not* atomic across the cluster.

**decode_responses is a whole-client setting.** It cannot be toggled per command. If part of your data is UTF-8 text and part is binary (e.g. pickled blobs or protobuf), a single decoded client will raise `UnicodeDecodeError` on the binary keys. Use separate clients, or leave it off and decode selectively.

**Redis vs Valkey.** After Redis Inc's 2024 license change (RSALv2/SSPLv1) and the Valkey fork, redis-py targets Redis-server semantics. It generally works against Valkey and other RESP servers, but new module features (search/JSON/etc.) assume Redis Stack / Redis 8 behavior.

## When to Use / When Not

**Use when:**
- You need the canonical, maintained Python client for Redis — this is the default choice.
- You want raw command access with sensible Python response types and no ORM overhead.
- You need sync and async against the same API surface, plus cluster/Sentinel/PubSub in one package.

**Avoid (or supplement) when:**
- You want object mapping / declarative models — reach for redis-om-python instead of hand-rolling on top of redis-py.
- You are on Valkey and want a client tracking that fork's roadmap specifically — a Valkey-aligned client may fit better.
- You need an async-first, fully type-annotated command surface — coredis is more opinionated there.

## Alternatives

- redis/redis-om-python — use when you want declarative models, indexing, and object mapping instead of raw commands.
- valkey-io/valkey-py — use when you target Valkey (the BSD-licensed Redis fork) and want a client aligned to it (it is itself a redis-py fork).
- alisaifee/coredis — use when you want an async-first client with fuller type annotations and typed command responses.
- aio-libs/aioredis — historical only; it was merged into redis-py's async module, so do not start new projects on it.
- redis/hiredis-py — not a client but the C parser; install it alongside redis-py when response-parsing throughput matters.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2009-11 | First release by Andy McCurdy[^1]. |
| 3.0.0 | 2018-10 | Dropped Python 2.6/legacy paths; response-encoding overhaul. |
| 4.0.0 | 2021-11 | `RedisCluster` merged in; asyncio client work begins (aioredis merge)[^3]. |
| 4.2.0 | 2022-03 | asyncio client shipped as `redis.asyncio`. |
| 5.0.0 | 2023-08 | RESP3 protocol support (`protocol=3`)[^2]. |
| 5.1 | 2024 | Drops Python 3.7; Python 3.8+ baseline. |
| 6.0.0 | 2025 | Client-side default search dialect (DIALECT 2); further RESP3 defaults. |
| 6.2.0 | 2026 | Python 3.9+ baseline; unified response shapes opt-in (`legacy_responses=False`). |

## References

[^1]: redis-py README, "Author" — developed and maintained by Redis Inc; original author Andy McCurdy. https://github.com/redis/redis-py#author
[^2]: redis-py README, "RESP3 Support" and unified responses migration guide. https://redis.readthedocs.io/en/stable/resp3_features.html
[^3]: aioredis / redis-py async merge notes. https://redis.readthedocs.io/en/stable/examples/asyncio_examples.html

## Tags

python, redis, database-client, key-value-store, cache, asyncio, connection-pool, redis-cluster, pubsub, resp3
