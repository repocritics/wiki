# redis/hiredis

> Minimalistic C client for Redis — a protocol codec and connection primitive, not a batteries-included driver.

[GitHub repo](https://github.com/redis/hiredis) ·
[License: BSD-3-Clause](https://github.com/redis/hiredis/blob/master/COPYING)

## Overview

Hiredis is the reference C client for Redis, in the main Redis GitHub
organization and maintained alongside the server[^1]. It is deliberately
small: it speaks the RESP wire protocol, exposes a `printf`-style command
formatter, and hands back parsed reply trees. It does not attempt to bind
every Redis command, manage a connection pool, or hide reconnection. That
minimalism is the whole design — the library is meant to be the lowest layer
that higher-level bindings and language clients build on.

Its most quietly influential piece is the **reply parser**, which is decoupled
from the I/O layer and shipped as a standalone stream parser (`read.c`)[^2].
That parser is reused far beyond this repo: `redis-cli`, many language
bindings, and downstream clients embed hiredis's RESP decoder rather than
reimplement it. When people say a Redis client is "built on hiredis," they
usually mean this parser plus the connection struct.

The defining tension is scope. Hiredis gives you one connection, one thread's
worth of state, and no opinion about clustering, failover, sharding, or
async concurrency beyond a thin event-loop hook. Everything past a single
blocking or non-blocking socket is left to you or to a wrapper library. This
keeps the core auditable and dependency-free, at the cost of a lot of
"assembly required" for production topologies.

## Getting Started

```bash
# Debian/Ubuntu
sudo apt-get install libhiredis-dev
# macOS
brew install hiredis
# or build from source
git clone https://github.com/redis/hiredis && cd hiredis && make && sudo make install
```

```c
#include <hiredis/hiredis.h>
#include <stdio.h>

int main(void) {
    redisContext *c = redisConnect("127.0.0.1", 6379);
    if (c == NULL || c->err) {
        fprintf(stderr, "connect: %s\n", c ? c->errstr : "alloc failed");
        return 1;
    }

    redisReply *r = redisCommand(c, "SET %s %b", "key", "val", (size_t)3);
    freeReplyObject(r);

    r = redisCommand(c, "GET %s", "key");
    if (r->type == REDIS_REPLY_STRING)
        printf("%s\n", r->str);   /* r->len holds the length */
    freeReplyObject(r);

    redisFree(c);
    return 0;
}
```

Compile with `-lhiredis`. Use `%b` (pointer + `size_t` length) for any
binary-safe value; `%s` calls `strlen` and stops at the first NUL.

## Architecture / How It Works

Hiredis is three loosely-coupled APIs over a common codec:

1. **Reply parser** (`read.c` / `read.h`) — a resumable RESP state machine.
   Feed it bytes, pull out `redisReply` trees. It has no socket dependency,
   which is why it is embedded elsewhere. It supports both RESP2 and the
   RESP3 types (map, set, double, bool, big number, verbatim string, push,
   attribute)[^3].
2. **Synchronous API** — `redisContext` holds a blocking socket plus an
   output buffer. `redisCommand` formats a command, appends it to the output
   buffer, flushes, and blocks on `redisGetReply`. Pipelining is exposed
   directly: `redisAppendCommand` fills the buffer without reading, then a
   sequence of `redisGetReply` calls drains replies, collapsing many commands
   into a single `write(2)` and `read(2)`.
3. **Asynchronous API** — `redisAsyncContext` wraps a non-blocking socket and
   dispatches per-command callbacks. It has no event loop of its own; instead
   it defines add-read / add-write / cleanup hooks that adapters bind to
   libev, libevent, libuv, glib, ivykis, or the macOS run loop. You pick the
   loop; hiredis only asks to be told when the fd is readable/writable.

The library vendors **sds** (simple dynamic strings), the same
length-prefixed string type used by Redis itself, for its internal buffers.
TLS is intentionally kept out of the core: OpenSSL support lives in a separate
`libhiredis_ssl` module so that a plain build has zero external dependencies.
A `redisContext` (and `redisAsyncContext`) is explicitly **not thread-safe** —
the concurrency unit is one context per thread.

## Production Notes

The differentiators here are the footguns, because the library trusts you.

- **Format-string injection.** `redisCommand(c, userInput)` treats its second
  argument as a format string. Passing untrusted data there is the C
  equivalent of SQL injection. Always use fixed format strings with `%s`/`%b`
  placeholders, or `redisCommandArgv` with explicit argument arrays.
- **Errors are terminal.** Once `c->err` is set, the context cannot be reused —
  you must `redisReconnect` or tear down and reconnect. There is no automatic
  recovery in the sync path.
- **Async frees replies for you.** Under the asynchronous API, hiredis calls
  `freeReplyObject` after your callback returns. Calling it yourself corrupts
  memory. This asymmetry with the sync API (where *you* free) is a common bug.
- **No pooling, no cluster, no Sentinel.** Hiredis is a single connection.
  Connection pools, Redis Cluster slot routing, and Sentinel-based failover
  are all out of scope; you need a wrapper (see Alternatives) or must build
  them yourself.
- **Security history.** CVE-2021-32765 was an integer overflow in the
  multi-bulk length handling that could corrupt memory on a hostile or
  malformed reply; it was fixed in 1.0.2, which is otherwise identical to
  1.0.0[^4]. Pin to >= 1.0.2. Note 1.0.1 erroneously bumped the SONAME and
  is skipped in the release line.
- **RESP3 doubles can be `nan`.** Since 1.1.0 a `REDIS_REPLY_DOUBLE` may hold
  `nan` in addition to `inf`/`-inf`; downstream parsing must account for it.
- **ABI.** The project tracks its ABI/API timeline publicly[^5]; the SONAME
  mishap aside, upgrades within a major line are generally recompile-only.

## When to Use / When Not

**Use when:**
- You are writing C/C++ and want a thin, auditable Redis connection.
- You are building a language binding or higher-level client and want a
  battle-tested RESP parser to embed.
- You need explicit control over pipelining, buffering, and the event loop.

**Avoid when:**
- You want pooling, cluster routing, Sentinel failover, or reconnection out of
  the box — reach for a wrapper instead.
- You want a memory-safe language experience; the raw C API is easy to misuse
  (format strings, manual `freeReplyObject`, terminal error state).
- You are not in C: use your language's native client, most of which already
  wrap or reimplement hiredis's parser.

## Alternatives

- sewenew/redis-plus-plus — use when you want a C++ API with connection
  pools, pipelining, transactions, cluster, and Sentinel built on top of
  hiredis.
- valkey-io/libvalkey — use when targeting Valkey (the Redis fork); it merges
  hiredis and hiredis-cluster into one actively maintained C client.
- Nordix/hiredis-cluster — use when you need Redis Cluster slot routing in C
  while staying close to the hiredis API.
- redis/redis — the server itself; `redis-cli` embeds hiredis but is not a
  library you link against.
- gomodule/redigo — use when you are in Go and want a native client rather
  than a C binding.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9.0 | 2011 | Major overhaul; introduced stateful `redisContext`[^2]. |
| 0.14.0 | 2020 | Stricter RESP length validation; `redisReply.len` became `size_t`. |
| 1.0.0 | 2020-08 | First stable release; RESP3, TLS module, bundled sds[^1]. |
| 1.0.2 | 2021-10 | Fix for CVE-2021-32765 (1.0.1 skipped, bad SONAME)[^4]. |
| 1.1.0 | 2023 | RESP3 doubles may return `nan`; assorted fixes. |
| 1.2.0 | 2024 | Additional `redisOptions` flags, keep-alive/timeout controls. |

## References

[^1]: Hiredis README and release notes, redis/hiredis. https://github.com/redis/hiredis
[^2]: Standalone reply parser, `read.c` / `read.h`. https://github.com/redis/hiredis/blob/master/read.c
[^3]: RESP3 specification (antirez/RESP3). https://github.com/antirez/RESP3/blob/master/spec.md
[^4]: CVE-2021-32765 security advisory, GHSA-hfm9-39pp-55p2. https://github.com/redis/hiredis/security/advisories/GHSA-hfm9-39pp-55p2
[^5]: Hiredis API/ABI timeline. https://abi-laboratory.pro/?view=timeline&l=hiredis

## Tags

c, redis, database-client, resp-protocol, networking, async-io, low-level, library, bsd-licensed, pipelining
