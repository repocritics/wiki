# tortoise/tortoise-orm

> An asyncio-native, Django-shaped ORM for Python — active-record models and relations without SQLAlchemy's abstraction depth.

[GitHub repo](https://github.com/tortoise/tortoise-orm) ·
[Official website](https://tortoise.github.io) ·
[License: Apache-2.0](https://github.com/tortoise/tortoise-orm/blob/develop/LICENSE.txt)

## Overview

Tortoise ORM is an `asyncio` Object-Relational Mapper for Python, first released in 2018[^1]. Its design goal is a familiar, Django-like API — models as classes, fields as class attributes, an active-record `.save()` / `.create()` surface, and a lazy `QuerySet` — but with every database operation being `await`-able rather than blocking[^2]. It targets CPython 3.10 and later and supports SQLite, PostgreSQL, MySQL/MariaDB, Microsoft SQL Server, and Oracle through pluggable async drivers[^2].

The project exists because the two obvious alternatives each have a gap. Django's ORM is synchronous at its core (async support is partial and wraps sync internals), and SQLAlchemy — while the mature, async-capable standard — exposes a deeper, more explicit abstraction (Core + Session/unit-of-work) than developers coming from Django expect. Tortoise aims at the middle: async-first, but with the "define a model, call `.filter()`, await the result" ergonomics of Django rather than SQLAlchemy's session and identity-map machinery.

The defining tension is maturity versus ergonomics. After years in production use Tortoise is still versioned pre-1.0 (0.x), its ecosystem is a fraction of SQLAlchemy's, and its open-issue count is large relative to project size — signals of finite maintainer bandwidth, not abandonment (the repository is actively pushed to[^3]). You adopt Tortoise for the clean async Django-like feel and accept a smaller ecosystem and less battle-tested edge behavior in return.

## Getting Started

```bash
pip install tortoise-orm             # SQLite (aiosqlite) only
pip install tortoise-orm[asyncpg]    # PostgreSQL
pip install tortoise-orm[asyncmy]    # MySQL/MariaDB
```

```python
from tortoise import Tortoise, fields, run_async
from tortoise.models import Model

class Tournament(Model):
    id = fields.IntField(primary_key=True)
    name = fields.CharField(max_length=100)

class Event(Model):
    id = fields.IntField(primary_key=True)
    name = fields.TextField()
    tournament = fields.ForeignKeyField(
        "models.Tournament", related_name="events",
        on_delete=fields.OnDelete.CASCADE,
    )

async def main():
    await Tortoise.init(db_url="sqlite://db.sqlite3",
                        modules={"models": ["__main__"]})
    await Tortoise.generate_schemas()   # dev only — use migrations in prod

    t = await Tournament.create(name="New")
    await Event.create(name="Final", tournament=t)

    # QuerySet is lazy — the query runs only when awaited
    events = await Event.filter(tournament__name="New").prefetch_related("tournament")
    for e in events:
        print(e.name, e.tournament.name)

run_async(main())
```

## Architecture / How It Works

Tortoise is an active-record ORM: model instances carry both data and persistence methods (`.save()`, `.delete()`), and `Model.filter(...)` returns a lazy `QuerySet` that emits SQL only when awaited, iterated, or sliced. SQL is generated through a fork of the PyPika query builder (`pypika-tortoise`) rather than hand-built strings, which is where dialect differences between Postgres, MySQL, and SQLite are absorbed.

The backend layer is a set of thin driver wrappers selected by the `db_url` scheme: `aiosqlite` for SQLite, `asyncpg` or `psycopg` for PostgreSQL, `aiomysql` or `asyncmy` for MySQL, `asyncodbc` for SQL Server/Oracle. Each maintains its own async connection pool. `Tortoise.init()` builds a **global** app/model registry and connection map — this global state is central to how the library is used and is the source of most of its testing friction.

Relations are first-class: `ForeignKeyField`, `OneToOneField`, and `ManyToManyField` (with an implicit or named through-table) generate reverse accessors automatically. Because relation traversal is I/O, Tortoise does not lazy-load across the sync boundary the way Django does — a related object that was not fetched cannot simply be dot-accessed and awaited implicitly. The intended pattern is `select_related()` (SQL JOIN, for FK/O2O) and `prefetch_related()` (separate batched queries, for reverse and M2M) to load relations up front and avoid N+1 query storms.

Supporting machinery: transactions via `in_transaction()` / the `@atomic()` decorator, signals (`pre_save`, `post_delete`, …), field-level and functional indexes, and a `contrib` layer with test helpers and — most consequentially for adoption — Pydantic integration (`pydantic_model_creator`) that turns models into request/response schemas, which is why Tortoise appears frequently in FastAPI stacks. Migrations were historically handled by the separate **Aerich** package; current releases ship a built-in migration framework and `tortoise` CLI (`makemigrations` / `migrate` / `sqlmigrate`)[^2].

## Production Notes

**`generate_schemas()` is not a migration system.** It creates tables on an empty database and is documented as development-only. Production schema evolution must go through the migration CLI (or Aerich on older setups); there is no automatic schema diffing at runtime.

**The relation-access footgun.** Accessing a related attribute that was not loaded does not transparently fetch it — it raises rather than issuing a hidden query, or requires an explicit `await instance.fetch_related(...)`. This is deliberate (no accidental N+1) but surprises developers arriving from Django. Forgetting `prefetch_related` / `select_related` in a loop is the most common performance regression.

**Global init and testing.** Because `Tortoise.init()` registers global connections, tests need disciplined setup/teardown. Use `tortoise.contrib.test` (or `initializer` / `finalizer`) rather than re-initializing ad hoc; parallel test workers sharing one SQLite file will contend.

**Driver choice matters.** `asyncmy` (Cython) is faster than `aiomysql` but requires a compiler / wheels at install time. For Postgres, `asyncpg` is the performance path; `psycopg` (v3) is the more conservative choice. Connection-pool sizing is per-driver and easy to under-provision under load.

**Pre-1.0 versioning.** The library remains on 0.x after years of use; minor version bumps have historically carried behavior changes. Pin the version and read release notes before upgrading rather than tracking latest.

**Ecosystem depth.** No admin site, a smaller plugin surface than SQLAlchemy, and fewer third-party recipes for exotic column types, composite keys, or advanced query patterns. Complex analytical SQL often ends up as raw queries via `.raw()` or `connection.execute_query()`.

## When to Use / When Not

**Use when:**
- You are building an async Python service (FastAPI, Sanic, Litestar) and want Django-like model ergonomics without a sync ORM bolted onto an event loop.
- Your schema is relational and CRUD-shaped, and you value readable models over query-engine control.
- You want tight Pydantic serialization for API request/response bodies.

**Avoid when:**
- You need the deepest, most battle-tested feature set — complex joins, composite primary keys, exotic dialect features, mature migrations at scale (choose SQLAlchemy).
- You are on a synchronous stack (WSGI/Flask without async) — Tortoise's async core buys you nothing there.
- You need a large ecosystem of integrations, an admin UI, or long-term API-stability guarantees.

## Alternatives

- sqlalchemy/sqlalchemy — the mature, async-capable standard; use it when you need breadth, composite keys, and a proven migration story (Alembic) over Django-like ergonomics.
- fastapi/sqlmodel — Pydantic models over SQLAlchemy Core; use when you want SQLAlchemy underneath but a lighter, typed model surface in FastAPI.
- collerek/ormar — Pydantic-native async ORM on SQLAlchemy Core; use when you want async models that *are* Pydantic schemas.
- piccolo-orm/piccolo — async ORM with an explicit query builder, playground, and admin; use when you want a first-class async admin UI.
- python-gino/gino — asyncpg-based async ORM; largely quiescent, consider only for legacy Postgres-only projects.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9.x | 2018 | First public releases; asyncio-native, Django-like model API[^1]. |
| 0.x (ongoing) | 2019–2024 | Multi-backend support (SQLite/Postgres/MySQL/MSSQL/Oracle), Pydantic contrib, signals, indexes. |
| current | 2026 | CPython 3.10+ requirement; built-in migration CLI; still pre-1.0[^2][^3]. |

## References

[^1]: Tortoise ORM repository, created 2018-03-29. https://github.com/tortoise/tortoise-orm
[^2]: Tortoise ORM README and documentation — features, supported databases, migration CLI. https://tortoise.github.io
[^3]: GitHub API repository metadata (stars, forks, license, last push) for tortoise/tortoise-orm, retrieved 2026-07. https://github.com/tortoise/tortoise-orm

## Tags

python, asyncio, orm, database, postgresql, mysql, sqlite, active-record, fastapi, pydantic, django-like
