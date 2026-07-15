# encode/databases

> Async connection layer that runs SQLAlchemy Core expressions against Postgres, MySQL, and SQLite — now archived by its maintainers.

[GitHub repo](https://github.com/encode/databases) ·
[Official website](https://www.encode.io/databases/) ·
[License: BSD-3-Clause](https://github.com/encode/databases/blob/master/LICENSE.md)

## Overview

`databases` is a small library from Encode (Tom Christie, the author of Starlette, HTTPX, and Django REST Framework) that gives asyncio code a single interface for talking to PostgreSQL, MySQL, and SQLite[^1]. It wraps the native async drivers — asyncpg, aiopg, aiomysql, asyncmy, aiosqlite — behind one `Database` object, and lets you build queries with the SQLAlchemy Core expression language rather than raw strings.

It arrived in 2019 to solve a specific, temporary problem: SQLAlchemy had no async story, so async web frameworks (Starlette, FastAPI, Sanic, Quart) had no first-class way to reach a relational database. `databases` filled that gap by reusing SQLAlchemy Core's dialects and expression compiler while supplying its own async execution and pooling on top of the async drivers. For roughly three years it was the default answer to "how do I use Postgres from FastAPI."

That defining tension is now its epitaph. SQLAlchemy 1.4 (2021) and 2.0 (2023) shipped native `async`/`await` support via `AsyncEngine` and `AsyncSession`, removing the reason `databases` existed. The Encode repository is **archived** (read-only) as of 2024[^2], with `0.9.0` as the final release. It still works and is still installed by many production apps, but it receives no fixes, and it never gained SQLAlchemy 2.0 support — it is pinned to the 1.4 line.

## Getting Started

```shell
$ pip install databases[aiosqlite]   # or [asyncpg], [aiomysql], [asyncmy], [aiopg]
```

```python
from databases import Database

database = Database("sqlite+aiosqlite:///example.db")
await database.connect()

# Bind parameters use :name style, regardless of the underlying driver.
await database.execute(
    "INSERT INTO scores(name, score) VALUES (:name, :score)",
    values={"name": "Daisy", "score": 92},
)

rows = await database.fetch_all("SELECT * FROM scores WHERE score > :min",
                                values={"min": 50})
for row in rows:
    print(row["name"], row["score"])   # Record behaves like a Mapping

await database.disconnect()
```

The URL scheme is `dialect+driver://…` — e.g. `postgresql+asyncpg://`, `mysql+aiomysql://`, `sqlite+aiosqlite://`. If you also use synchronous SQLAlchemy (for `create_all()` or Alembic migrations), you must additionally install a sync driver such as psycopg2 or PyMySQL[^1].

## Architecture / How It Works

`databases` is deliberately thin. It does **not** use the SQLAlchemy ORM, engine, or connection pool. It borrows only SQLAlchemy Core's *dialects* (to know each backend's SQL flavor) and its *expression compiler* (to turn `sqlalchemy.select(...)` / `table.insert()` constructs into a SQL string plus bound parameters). Execution and pooling are then delegated to the raw async driver for that backend[^3].

- **Query methods**: `fetch_all`, `fetch_one`, `fetch_val`, `execute`, `execute_many`, and `iterate` (a streaming async generator). Results are `Record` objects — read-only, `Mapping`-like, index- or key-accessible.
- **Parameter binding**: queries use named `:param` placeholders. The Core compiler rewrites these into whatever the driver expects (asyncpg's `$1` positional form, for instance), so the same query string is portable across backends.
- **Transactions**: `async with database.transaction():` opens a transaction; nested `async with` blocks map to savepoints. A `@database.transaction()` decorator form also exists.
- **Connection management**: connections are tracked per-task via `contextvars`, so concurrent tasks each get their own connection from the driver's pool. Pool sizing (`min_size`, `max_size`) is passed through the URL query string or `Database(...)` kwargs.
- **Test affordance**: `Database(url, force_rollback=True)` wraps the whole connection in a transaction that is rolled back on disconnect — a common pattern for isolating test cases.

The coupling story matters: because query building is SQLAlchemy Core but execution is not SQLAlchemy, you get Core's expressive, dialect-aware query construction without Core's engine, events, or 2.0 features. That seam is exactly where the library froze — it tracks SQLAlchemy 1.4's Core API and cannot move to 2.0 without a rewrite that never happened.

## Production Notes

- **It is archived and unmaintained.** This is the dominant operational fact. No CVE fixes, no driver-compatibility updates, no new Python version support beyond what shipped in `0.9.0`. Treat new adoption as taking on frozen code, and plan a migration path for existing use.
- **Pinned to SQLAlchemy 1.4.** `databases` requires `sqlalchemy>=1.4.42,<2.0`. In a dependency tree where anything else wants SQLAlchemy 2.0, this becomes an unresolvable conflict and is the single most common reason teams are forced off the library.
- **Not an ORM.** No models, relationships, lazy loading, or identity map. You write Core expressions or raw SQL yourself. This is a feature for people who want it and a surprise for people expecting Django/SQLAlchemy-ORM ergonomics.
- **Raw-string SQL is still your responsibility.** The `:param` binding is safe, but any value you interpolate into the query string yourself (table names, f-strings) is an injection surface. Always pass user data through `values=`.
- **Driver quirks leak through.** Because execution is the raw driver, backend-specific behavior (asyncpg's strict type coercion, its lack of implicit `str→int` casts, prepared-statement caching interacting with PgBouncer in transaction-pooling mode) surfaces directly. The abstraction is over query *syntax*, not driver *semantics*.
- **SQLite is single-writer.** `aiosqlite` runs on a background thread; it is fine for tests and low-concurrency apps but is not a concurrent-write store. Don't benchmark against it and assume Postgres behavior.
- **Connection lifecycle in ASGI.** With FastAPI/Starlette, call `connect()` on startup and `disconnect()` on shutdown; forgetting the latter leaks pooled connections across reloads in development.

## When to Use / When Not

**Use when:**
- You are maintaining an existing app already built on `databases` and it works — there is no urgency to rip it out until a dependency forces SQLAlchemy 2.0.
- You want Core-style query building with async drivers and are comfortable pinning to SQLAlchemy 1.4 for the life of the project.
- You need a very small, well-understood surface over asyncpg/aiomysql/aiosqlite and don't want an ORM.

**Avoid when:**
- You are starting something new — the native SQLAlchemy 2.0 async layer covers the same ground and is maintained.
- You need SQLAlchemy 2.0, an ORM, or long-term security support.
- Your team expects an actively developed dependency; an archived repo is a supply-chain and compliance liability.

## Alternatives

- sqlalchemy/sqlalchemy — since 1.4, `create_async_engine` + `AsyncSession` provide native async over the same drivers; use this for anything new, and as the migration target off `databases`.
- MagicStack/asyncpg — use directly when you want maximum Postgres throughput and don't need cross-backend portability or Core query building.
- tortoise/tortoise-orm — use when you want a Django-like async ORM with models and migrations rather than a query layer.
- collerek/ormar — use when you want an async ORM that itself sits on SQLAlchemy Core, with a Pydantic-friendly model API.
- piccolo-orm/piccolo — use when you want a fully async ORM and query builder designed for asyncio from the start.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2019-02 | Async layer over SQLAlchemy Core for Starlette-era frameworks[^1]. |
| 0.4.x | 2020 | Broad framework adoption alongside the FastAPI boom. |
| 0.6.0 | 2022 | Migrated to the SQLAlchemy 1.4 Core API; dropped older 1.3 support[^3]. |
| 0.7–0.8 | 2023 | Driver/compatibility maintenance releases. |
| 0.9.0 | 2024 | Final release. Repository archived (read-only)[^2]. |

## References

[^1]: Databases documentation and README, Encode. https://www.encode.io/databases/
[^2]: encode/databases repository — marked archived (read-only) by the maintainers; last push 2024-05-21. https://github.com/encode/databases
[^3]: Databases source and SQLAlchemy Core dependency (`sqlalchemy>=1.4.42,<2.0`), PyPI project page. https://pypi.org/project/databases/

## Tags

python, asyncio, database, sqlalchemy, postgresql, mysql, sqlite, orm-alternative, query-builder, archived, asgi
