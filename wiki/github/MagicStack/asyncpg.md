# MagicStack/asyncpg

> A PostgreSQL client for Python/asyncio that talks the wire protocol directly instead of wrapping libpq or the DB-API.

[GitHub repo](https://github.com/MagicStack/asyncpg) ·
[Documentation](https://magicstack.github.io/asyncpg/current/) ·
[License: Apache-2.0](https://github.com/MagicStack/asyncpg/blob/master/LICENSE)

## Overview

asyncpg is an asyncio-native PostgreSQL driver from MagicStack, the team behind uvloop. It was created by Yury Selivanov (a CPython core developer and co-author of the `async`/`await` syntax) and first released in 2016[^1]. Its distinguishing decision is that it does not use libpq and does not implement the DB-API (PEP 249). Instead it reimplements the PostgreSQL frontend/backend binary protocol from scratch in Cython, decoding rows directly into Python objects with per-type codecs[^2].

That choice is the whole story of the library. Bypassing libpq and the text protocol is what makes asyncpg fast — MagicStack's own benchmark (June 2023, geometric mean) puts it around 5x psycopg3[^3] — but it also means asyncpg is its own API, not a drop-in for the psycopg ecosystem. Parameters are PostgreSQL-native positional placeholders (`$1`, `$2`), not `%s`; rows come back as `Record` objects; and there is no `cursor()`/`execute()` DB-API surface. Code and tooling written against DB-API drivers do not port unchanged.

The other defining trait is that asyncpg prepares statements aggressively. Nearly every query is turned into a server-side prepared statement and cached, which is where most of its speed and most of its operational surprises come from (see Production Notes). After roughly a decade of development the project is still on 0.x versioning; the latest release is 0.31.0 (November 2025)[^4], and it is maintained but not fast-moving.

## Getting Started

```bash
pip install asyncpg
# GSSAPI/SSPI (Kerberos) auth needs an extra:
pip install 'asyncpg[gssauth]'
```

asyncpg requires Python 3.9+ and targets PostgreSQL 9.5 through 18[^1]. With no GSSAPI it has no runtime dependencies.

```python
import asyncio
import asyncpg

async def main():
    # A single connection
    conn = await asyncpg.connect(
        user="user", password="password",
        database="database", host="127.0.0.1",
    )
    # Native positional params: $1, not %s
    rows = await conn.fetch("SELECT id, name FROM users WHERE id = $1", 10)
    for r in rows:              # r is an asyncpg.Record (tuple + mapping)
        print(r["id"], r["name"])
    await conn.close()

asyncio.run(main())
```

For anything concurrent, use the built-in pool rather than raw connections:

```python
async def main():
    pool = await asyncpg.create_pool(dsn="postgresql://user:pw@localhost/db",
                                     min_size=5, max_size=20)
    async with pool.acquire() as conn:
        val = await conn.fetchval("SELECT count(*) FROM users")
    await pool.close()
```

## Architecture / How It Works

**No libpq.** asyncpg speaks the PostgreSQL protocol itself. The protocol codec, the connection state machine, and the row-decoding hot paths are written in Cython and compiled to C, which is why wheels are platform-specific and a source install needs a C compiler. Data is transferred in PostgreSQL's binary format where possible, and each PostgreSQL type maps to a codec that decodes bytes straight into a Python object — no intermediate text parsing[^2].

**Records, not dicts.** Query results are `asyncpg.Record` instances: an immutable object that indexes both positionally (`r[0]`) and by column name (`r["name"]`), and supports `dict(r)`. This avoids building a dict per row but means downstream code expecting plain dicts needs an explicit conversion.

**Prepared-statement first.** `fetch`, `fetchrow`, `fetchval`, and `execute` implicitly Parse/Bind/Execute a server-side prepared statement, and asyncpg keeps an LRU cache of these per connection (`statement_cache_size`, default 100). Repeated queries skip the Parse step and reuse the server's cached plan. This is the main source of throughput and the main source of the footguns below.

**Type system.** Composite types, arrays, ranges, enums, and combinations are decoded automatically by introspecting the catalog. Custom encode/decode is registered per-connection with `set_type_codec` / `set_builtin_type_codec` (a common pattern for `jsonb`, `numeric` as `float`, or PostGIS geometry). Because codecs are introspected and cached on the connection, they must be re-registered on every new connection — typically via the pool's `init` callback.

**Pool.** `create_pool` gives a fixed-range pool of connections with acquire/release, an optional `setup`/`init` hook, and connection recycling. There is no cross-process pooling; it is an in-process asyncio pool.

**No DB-API layer.** asyncpg deliberately does not implement PEP 249. ORMs and query builders reach it through adapters: SQLAlchemy ships an `asyncpg` dialect, Databases and encode/`databases` wrap it, and Piccolo/Tortoise use it as a backend. Direct SQL is the intended interface.

## Production Notes

**PgBouncer in transaction or statement pooling mode is the classic failure.** Because asyncpg caches prepared statements on a connection, and transaction-mode PgBouncer hands a different physical connection to each transaction, cached statement names collide or vanish and you get errors like `prepared statement "__asyncpg_stmt_x__" does not exist`. The historical fix is to disable both caches: `statement_cache_size=0` and `max_cached_statement_lifetime=0` — which also gives up asyncpg's main performance advantage. Newer setups can instead use PgBouncer 1.21+'s native prepared-statement support in transaction mode, but this must be configured deliberately[^5].

**Cached plan invalidation after DDL.** A long-lived connection holds prepared statements whose server-side plans reference specific table/column OIDs. Run a migration (`ALTER TABLE`, add/drop a column, change a type) while connections stay open and subsequent queries fail with `cached plan must not change result type` / `cached statement plan is invalid`. The pool does not automatically flush statement caches on schema change; deployments that migrate live typically recycle the pool or set a `max_cached_statement_lifetime`.

**Not thread-safe and not multiprocess-shared.** A connection is bound to one event loop. Do not share a connection or pool across threads, processes, or event loops. With Gunicorn/Uvicorn workers, each worker needs its own pool created inside its own loop.

**`Record` gotchas.** Records are not dicts and not JSON-serializable directly; `json.dumps(row)` fails until you `dict(row)`. `numeric`/`decimal` columns decode to `decimal.Decimal`, and `jsonb` decodes to a `str` unless you register a codec that runs `json.loads`. These are frequent first-day surprises.

**Codec registration is per-connection.** Any `set_type_codec` call applies only to the connection it was made on. Register codecs in the pool's `init` coroutine, or new pool connections silently revert to defaults.

**Parameter style and casting.** Placeholders are `$1..$n` and asyncpg is stricter about types than text-protocol drivers; ambiguous cases sometimes need an explicit `$1::int` cast in the SQL. There is no client-side string interpolation of parameters — which is good for injection safety but means dynamic identifiers still require care.

**uvloop pairing.** asyncpg's benchmarks assume uvloop as the event loop. On the default asyncio loop it is still fast, but the headline numbers come from the uvloop + asyncpg combination from the same authors.

## When to Use / When Not

**Use when:**
- You are PostgreSQL-only and on asyncio, and want the fastest mainstream driver.
- You write SQL directly (or use SQLAlchemy's async engine) and want prepared statements, binary decoding, and native composite/array support.
- You are throughput-bound on database round-trips and can pair it with uvloop.
- You need PostgreSQL-specific features (LISTEN/NOTIFY, COPY, scrollable cursors) exposed rather than hidden.

**Avoid when:**
- You sit behind PgBouncer in transaction/statement mode and cannot configure prepared statements — you will fight the statement cache.
- You need DB-API compatibility or a synchronous driver — asyncpg is neither.
- You need multi-database portability; asyncpg is PostgreSQL and nothing else.
- Your workload is light and psycopg's ubiquity, DB-API tooling, and simpler pooling story matter more than raw speed.

## Alternatives

- psycopg/psycopg — the modern (psycopg3) reference PostgreSQL driver; DB-API compliant, sync and async, libpq-based. Use instead when you want compatibility and ecosystem breadth over peak async throughput.
- psycopg2 — the legacy sync workhorse; use when maintaining existing sync codebases or tooling that assumes it.
- aiopg — asyncio wrapper over psycopg2. Largely superseded; use only for legacy code that already depends on it.
- sqlalchemy/sqlalchemy — not a driver but the usual layer above; its async engine drives asyncpg under the hood. Use when you want an ORM/core query builder rather than raw SQL.
- pgjdbc-style or `databases` (encode/databases) — a thin async wrapper offering a uniform API across asyncpg and others; use when you want backend-swappable async SQL.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.6.0 | 2016-10-20 | Early public release[^1]. |
| 0.11.0 | 2017-05-11 | Pool and API maturation. |
| 0.15.0 | 2018-02-15 | Continued protocol/codec work. |
| 0.18.0 | 2018-10-30 | Broad type and feature coverage. |
| 0.20.0 | 2019-11-21 | Stability release. |
| 0.22.0 | 2021-02-10 | Newer PostgreSQL and Python support. |
| 0.25.0 | 2021-11-16 | Ongoing maintenance. |
| 0.27.0 | 2022-10-26 | PostgreSQL 15-era support. |
| 0.29.0 | 2023-11-05 | Python 3.12 support. |
| 0.30.0 | 2024-10-20 | Python 3.13 support. |
| 0.31.0 | 2025-11-24 | Latest release[^4]. |

## References

[^1]: asyncpg README and documentation — requirements (Python 3.9+, PostgreSQL 9.5–18) and history. https://magicstack.github.io/asyncpg/current/
[^2]: asyncpg documentation, "Introduction" / internals — native binary protocol implementation and per-type codecs. https://magicstack.github.io/asyncpg/current/index.html
[^3]: MagicStack, "1M rows from Postgres to Python" and the pgbench benchmark toolset (June 2023 results, ~5x psycopg3). http://magic.io/blog/asyncpg-1m-rows-from-postgres-to-python/ · https://github.com/MagicStack/pgbench
[^4]: asyncpg GitHub releases — v0.31.0 published 2025-11-24. https://github.com/MagicStack/asyncpg/releases
[^5]: asyncpg FAQ, "Why am I getting prepared statement errors?" (PgBouncer transaction pooling and `statement_cache_size`). https://magicstack.github.io/asyncpg/current/faq.html

## Tags

python, asyncio, postgresql, database-driver, async, cython, prepared-statements, connection-pool, sql, high-performance
