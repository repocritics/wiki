# sqlalchemy/sqlalchemy

> The Python SQL toolkit and ORM that refuses to hide the relational database behind objects.

[GitHub repo](https://github.com/sqlalchemy/sqlalchemy) ·
[Official website](https://www.sqlalchemy.org) ·
[License: MIT](https://github.com/sqlalchemy/sqlalchemy/blob/main/LICENSE)

## Overview

SQLAlchemy is a two-layer database toolkit for Python: a **Core** SQL expression language and connection/pooling layer, and an **ORM** built on top of it. It was created by Michael Bayer, with the first release in 2006[^1]; the ORM is a from-scratch implementation of the data mapper, unit-of-work, and identity-map patterns rather than the active-record pattern used by Django and Rails[^2]. As of 2026 it remains the default answer for non-trivial relational database access in Python — the layer underneath FastAPI apps, Flask apps, data pipelines, and most of the Python web ecosystem that is not committed to Django's built-in ORM.

The project's defining stance is stated in its own README: "an ORM doesn't need to hide the R." SQLAlchemy deliberately exposes joins, subqueries, correlation, and set-based operations instead of pretending a database is an object collection. This is its strength (you never lose access to real SQL) and its cost (the learning curve is steep, and the ORM's Session/identity-map model has more moving parts than active-record ORMs). The Core layer is usable entirely on its own, and many teams use it without ever touching the ORM.

The other structural fact to understand is the **1.4 → 2.0 transition**. SQLAlchemy 2.0 (2023) unified the Core and ORM query APIs around a single `select()` construct, made PEP 484 typing first-class via `Mapped[]` annotations, and deprecated the legacy `Query` object and implicit autocommit[^3]. Code and tutorials written before 2021 use a materially different API surface, which is the single biggest source of confusion for people learning it today.

## Getting Started

```bash
pip install SQLAlchemy
# with a driver, e.g. PostgreSQL:
pip install SQLAlchemy psycopg2-binary
# async extra pulls in greenlet:
pip install "SQLAlchemy[asyncio]"
```

```python
# 2.0-style declarative ORM
from sqlalchemy import create_engine, select, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))

engine = create_engine("sqlite:///app.db")
Base.metadata.create_all(engine)

with Session(engine) as session:
    session.add(User(name="Tom"))
    session.commit()

    stmt = select(User).where(User.name == "Tom")
    user = session.scalars(stmt).one()
    print(user.id, user.name)
```

The same `select()` construct works in Core without the ORM, operating on `Table` metadata and returning row tuples instead of mapped objects.

## Architecture / How It Works

The stack is layered, and the layers are genuinely separable:

1. **DBAPI + dialects** — At the bottom is Python's DBAPI (PEP 249). SQLAlchemy wraps each driver in a *dialect* that knows the SQL flavor and type quirks of a backend: `psycopg2`/`psycopg`/`asyncpg` for PostgreSQL, `pymysql`/`aiomysql` for MySQL, `sqlite3` for SQLite, `oracledb` for Oracle, `pyodbc` for SQL Server. Swapping databases is mostly a URL change plus awareness of dialect-specific behavior.
2. **Connection pool** — `QueuePool` by default. It holds a fixed set of connections (`pool_size`) plus an overflow allowance (`max_overflow`), and hands them out per-checkout. This is why a single `Engine` is meant to be a long-lived, module-level object, not created per request.
3. **Core / SQL expression language** — A Python object graph (`select()`, `Table`, `Column`, `and_()`, `func.*`) that compiles to backend-specific SQL strings with bound parameters. Everything renders through bind parameters, so literal SQL injection is structurally avoided. Schema metadata can be *reflected* from an existing database or used to emit `CREATE` statements.
4. **ORM** — Adds the `Session`, which implements the identity map (one Python object per primary key per session) and unit of work (changes are batched and flushed as ordered `INSERT`/`UPDATE`/`DELETE` on `commit()` or `flush()`). Relationships between mapped classes are configured with `relationship()` and loaded via strategies.

**Async** support (added in 1.4) is implemented by running the synchronous internals inside a `greenlet` and exposing `AsyncEngine` / `AsyncSession`[^4]. It is not a native async rewrite — it is the sync core executed under a greenlet bridge — which is why async lazy-loading requires special handling rather than working transparently.

**Alembic** (a separate project, also by Michael Bayer) is the migrations tool that pairs with SQLAlchemy metadata; SQLAlchemy itself does not do schema migrations[^5].

## Production Notes

**Lazy loading is the classic footgun.** By default, accessing a `relationship()` attribute emits a query on access. Iterate a list of parents and touch a child relationship inside the loop and you get the N+1 problem — one query per row. Fix it with explicit eager-loading strategies: `selectinload()` (one extra `IN` query, usually the right default), `joinedload()` (single JOIN, can multiply rows), or `subqueryload()`.

**DetachedInstanceError.** If you access a lazy attribute on an object after its `Session` has closed, you get `DetachedInstanceError`. Combined with `expire_on_commit=True` (the default), attributes are expired after `commit()` and re-fetched on next access — which fails if the session is gone. The standard fixes are keeping the session open for the object's lifetime, eager-loading before close, or setting `expire_on_commit=False`.

**Session scope.** The Session is not thread-safe and is meant to be short-lived — typically one per web request or per unit of work. Long-lived or shared sessions accumulate identity-map state and cause stale reads. Web frameworks use `scoped_session` or dependency-injected per-request sessions.

**Pool tuning under load.** `pool_size` + `max_overflow` caps concurrent connections; exceed it and checkouts block up to `pool_timeout` then raise `TimeoutError`. Behind PgBouncer in transaction-pooling mode, disable driver-side prepared-statement caching. Set `pool_pre_ping=True` to survive database restarts and idle-connection drops instead of getting a stale-connection error on first use.

**The 1.x → 2.0 upgrade.** The migration path is real work: the legacy `Query` API (`session.query(User)...`) still functions but is deprecated in favor of `select()`; implicit autocommit was removed; `Mapped[]`-annotated declarative replaces the older `Column`-attribute style. SQLAlchemy provides `SQLALCHEMY_WARN_20=1` to surface every deprecated call site before you flip the version[^3]. Budget for it; do not treat it as a patch bump.

**Typing.** 2.0's `Mapped[]` annotations give real static types without the old Mypy plugin, but generic queries and hybrid properties still occasionally defeat type checkers. It is far better than pre-2.0, not perfect.

## When to Use / When Not

**Use when:**
- You need full, explicit SQL control while still getting object mapping and change tracking.
- You want to target multiple databases (dev on SQLite, prod on PostgreSQL) with the same code.
- You want the Core layer as a SQL builder without any ORM at all.
- You are outside Django and need a mature, well-supported ORM.

**Avoid when:**
- You are already in Django — its built-in ORM is tightly integrated and switching buys little.
- You want an active-record, convention-over-configuration ORM with minimal ceremony (SQLAlchemy's explicitness is overhead for small CRUD apps).
- You want a natively-async ORM and don't want the greenlet bridge's sharp edges around lazy loading.
- Your workload is a handful of raw queries — a driver like `asyncpg` plus hand-written SQL is simpler.

## Alternatives

- tiangolo/sqlmodel — SQLAlchemy + Pydantic in one model class; use when you're on FastAPI and want request/response and DB models to share a definition.
- django/django — use its built-in ORM instead when your app is already a Django app.
- tortoise/tortoise-orm — use when you want a Django-style, natively-async ORM without the greenlet bridge.
- coleifer/peewee — use for small apps that want a lighter, simpler ORM with less surface area.
- MagicStack/asyncpg — use when you don't need an ORM at all and want the fastest raw async PostgreSQL driver.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2006-02 | First public release, by Michael Bayer[^1]. |
| 0.5 | 2009-01 | Session/unit-of-work maturation. |
| 1.0 | 2015-04 | Performance-focused release[^6]. |
| 1.1–1.3 | 2016–2019 | JSON types, baked queries, incremental Core/ORM work. |
| 1.4 | 2021-03 | 2.0-style APIs behind a `future` flag; asyncio via greenlet[^4]. |
| 2.0 | 2023-01 | Unified `select()` API, `Mapped[]` typing, `Query`/autocommit deprecated[^3]. |

## References

[^1]: Michael Bayer is the original author and lead maintainer; the project dates to 2006. Project history and philosophy: https://www.sqlalchemy.org/features.html
[^2]: SQLAlchemy ORM overview — data mapper, unit of work, identity map. https://docs.sqlalchemy.org/en/20/orm/
[^3]: SQLAlchemy 2.0 migration guide (`select()` unification, `Mapped[]` typing, `SQLALCHEMY_WARN_20`). https://docs.sqlalchemy.org/en/20/changelog/migration_20.html
[^4]: SQLAlchemy asyncio extension (AsyncEngine/AsyncSession, greenlet bridge). https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
[^5]: Alembic — database migrations for SQLAlchemy. https://alembic.sqlalchemy.org/
[^6]: SQLAlchemy changelog. https://docs.sqlalchemy.org/en/20/changelog/

## Tags

python, orm, sql, database, data-mapper, connection-pooling, postgresql, mysql, sqlite, asyncio, relational
