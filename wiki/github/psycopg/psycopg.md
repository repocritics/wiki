# psycopg/psycopg

> Psycopg 3 — the ground-up rewrite of Python's most-used PostgreSQL adapter, with native async and server-side parameter binding.

[GitHub repo](https://github.com/psycopg/psycopg) ·
[Official website](https://www.psycopg.org/psycopg3/) ·
[License: LGPL-3.0](https://github.com/psycopg/psycopg/blob/master/LICENSE.txt)

## Overview

This repository is **Psycopg 3**, a modern PostgreSQL database adapter for Python maintained by Daniele Varrazzo[^1]. It is the successor to the widely deployed **psycopg2**, which lives in a separate repository (`psycopg/psycopg2`) and is still the more-installed of the two. Psycopg 3 is not a drop-in replacement: it keeps the DB-API 2.0 shape and much of the surface familiar to psycopg2 users, but changes several defaults and internals deliberately[^2].

The defining feature is that Psycopg 3 was written from scratch to support **both synchronous and asyncio** code paths from the same codebase (`psycopg.connect()` vs `psycopg.AsyncConnection.connect()`), and to use PostgreSQL's **extended query protocol with server-side parameter binding by default**. In psycopg2, query parameters were interpolated client-side into the SQL string; in Psycopg 3 the query and its parameters are sent separately to the server[^3]. This is safer and enables prepared statements and pipeline mode, but it is also the single biggest source of migration surprises.

Psycopg 3 is a thin, correct layer over **libpq**, the official PostgreSQL C client library, rather than a reimplementation of the wire protocol. That coupling means it inherits libpq's behavior, SSL handling, connection-string parsing, and version support, at the cost of requiring libpq to be present on the host. Its main audience is applications and frameworks (Django 4.2+, SQLAlchemy) that need a maintained, standards-compliant driver with a real async story.

## Getting Started

```bash
pip install "psycopg[binary,pool]"
```

`binary` pulls a self-contained libpq wheel (convenient, but the maintainer recommends building against the system libpq for production); `pool` adds the connection-pool package.

```python
import psycopg

# Sync — context managers commit/close automatically
with psycopg.connect("dbname=test user=postgres") as conn:
    with conn.cursor() as cur:
        cur.execute(
            "INSERT INTO numbers (n) VALUES (%s)",
            (100,),                       # server-side bound, not string-interpolated
        )
        cur.execute("SELECT n FROM numbers WHERE n > %s", (50,))
        print(cur.fetchall())
```

```python
import asyncio
import psycopg

async def main():
    async with await psycopg.AsyncConnection.connect("dbname=test") as conn:
        async with conn.cursor() as cur:
            await cur.execute("SELECT now()")
            print(await cur.fetchone())

asyncio.run(main())
```

## Architecture / How It Works

Psycopg 3 ships as **three separately released packages** so they can version independently[^2]:

1. **`psycopg`** — the pure-Python implementation. Runtime dependency: libpq.
2. **`psycopg_c`** — an optional C/Cython speedup module. When installed it is used automatically; otherwise the pure-Python path runs.
3. **`psycopg_pool`** — sync and async connection pools, kept separate to allow a different release cadence.

`pip install "psycopg[binary]"` swaps in a **prebuilt libpq wheel** instead of linking the system one; `pip install psycopg` alone links against whatever libpq is on the host.

**Parameter binding.** Queries use `%s` (positional) or `%(name)s` (named) placeholders. The query text and the parameter tuple are passed to `PQexecParams`/prepared-statement APIs so PostgreSQL binds them — no client-side quoting. Composing dynamic SQL (identifiers, snippets) is done with the `psycopg.sql` module (`sql.SQL`, `sql.Identifier`), not f-strings.

**Type adaptation** goes through *dumpers* (Python → PostgreSQL) and *loaders* (PostgreSQL → Python), registered per-connection or globally in a `TypeInfo`/`AdaptersMap`. Postgres arrays, composites, ranges, JSON/JSONB, and enums are handled through this system; custom types register their own adapters.

**Prepared statements** are created automatically once a query has been executed enough times (`prepare_threshold`, default 5), then reused. **Pipeline mode** (`conn.pipeline()`) batches multiple statements without waiting for each round-trip — a libpq feature psycopg2 never exposed. **COPY** has a first-class API (`cursor.copy()`) that streams rows in and out efficiently.

## Production Notes

**Server-side binding breaks psycopg2 idioms.** The change from client-side interpolation is the top migration footgun[^3]:
- You cannot send **multiple semicolon-separated statements** in one `execute()` when using parameters.
- A placeholder cannot be used for an identifier, a keyword, or inside a string literal — only for a value. Use `psycopg.sql` for the rest.
- Some queries need an explicit cast (e.g. `%s::int`) because the server, not the client, now infers parameter types, and it occasionally cannot.
- A literal `%` in SQL must be written `%%` when parameters are passed.

**The `binary` package is discouraged for production.** The bundled-libpq wheel is convenient for getting started and for CI, but the docs recommend installing `psycopg` (source) against a system libpq so you get the platform's SSL/GSS/version behavior and security updates[^4]. The binary build also cannot use some libpq features that depend on how it was compiled.

**Connection pool must be opened explicitly.** `ConnectionPool(...)` historically opened in its constructor; that path is deprecated and warns — you are expected to call `pool.open()` / use it as a context manager, and to size `min_size`/`max_size` for your workload[^5]. The async pool is `AsyncConnectionPool`. A pool that is never opened, or opened but never `close()`d, is a common leak.

**Autocommit is off by default** (DB-API). Every connection starts a transaction on first execute; forgetting to `commit()` (or not using the connection as a context manager, which commits on clean exit) leaves changes invisible and can hold locks. Set `autocommit=True` for DDL-heavy or one-shot work.

**Performance.** Install `psycopg[c]` or `psycopg_c` in production — the pure-Python path is meaningfully slower for row-heavy workloads. For very high-throughput async services, `asyncpg` is faster still because it skips libpq and speaks the protocol directly; Psycopg 3 trades some of that speed for libpq compatibility and a unified sync/async API.

**Coexistence with psycopg2.** The two packages install side by side (`import psycopg` vs `import psycopg2`). Django uses psycopg 3 automatically when it is installed on Django 4.2+, so an incidental `pip install psycopg` can switch your ORM's driver — pin deliberately[^6].

## When to Use / When Not

**Use when:**
- You want one adapter that serves both sync and asyncio code with the same API.
- You need pipeline mode, first-class COPY, or automatic prepared statements.
- You're on Django 4.2+/SQLAlchemy 2.0 and want the maintained, current driver.
- You want to rely on the official libpq (SSL, connection strings, auth methods) rather than a reimplementation.

**Avoid when:**
- You have a large psycopg2 codebase and no capacity to audit for server-side-binding breaks — psycopg2 is still supported.
- You need the absolute fastest async Postgres throughput and don't need libpq — asyncpg wins on raw speed.
- You cannot ship libpq on the host and won't use the binary wheel.
- You depend on a psycopg2-only extension or a library that hard-imports `psycopg2`.

## Alternatives

- psycopg/psycopg2 — the predecessor; use it when you need maximum ecosystem/legacy compatibility and don't need native async.
- MagicStack/asyncpg — async-only, no libpq, faster raw throughput; use it when async performance is the priority and DB-API compatibility is not.
- sqlalchemy/sqlalchemy — a toolkit/ORM, not a driver; use it when you want query building and mapping on top (it drives psycopg or asyncpg underneath).
- tlocke/pg8000 — pure-Python driver with no libpq dependency; use it when you cannot install native libraries.
- aio-libs/aiopg — asyncio wrapper over psycopg2; largely superseded by Psycopg 3's native async, use only for existing aiopg code.

## History

| Version | Date | Notes |
|---------|------|-------|
| psycopg2 1.0 | ~2006 | The predecessor line (separate repo), still the most-installed. |
| 3.0 | 2021-10 | First stable release of the from-scratch rewrite; native async, server-side binding[^1]. |
| 3.1 | 2022-09 | Client-side-binding cursors, connection-pool refinements, prepared-statement tuning. |
| 3.2 | 2024-06 | Pool API changes (explicit open), raw/scalar rows, capabilities checks, `pgvector`-friendly typing. |

## References

[^1]: Psycopg 3 documentation and project home. https://www.psycopg.org/psycopg3/
[^2]: Psycopg 3 installation — package layout (`psycopg`, `psycopg_c`, `psycopg_pool`). https://www.psycopg.org/psycopg3/docs/basic/install.html
[^3]: "Differences from `psycopg2`" — server-side binding, multiple statements, placeholders. https://www.psycopg.org/psycopg3/docs/basic/from_pg2.html
[^4]: Psycopg 3 — binary vs local installation guidance. https://www.psycopg.org/psycopg3/docs/basic/install.html#binary-installation
[^5]: Psycopg 3 connection pools. https://www.psycopg.org/psycopg3/docs/advanced/pool.html
[^6]: Django 4.2 release notes — psycopg 3 support. https://docs.djangoproject.com/en/stable/releases/4.2/

## Tags

python, postgresql, database, database-driver, db-api, asyncio, libpq, sql, orm-backend, connection-pool
