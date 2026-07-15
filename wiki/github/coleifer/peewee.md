# coleifer/peewee

> A small, single-file Python ORM that exposes SQL directly rather than hiding it — expressive query building without the machinery of a unit-of-work.

[GitHub repo](https://github.com/coleifer/peewee) ·
[Official website](http://docs.peewee-orm.com/) ·
[License: MIT](https://github.com/coleifer/peewee/blob/master/LICENSE)

## Overview

Peewee is a Python ORM written and maintained almost single-handedly by Charles Leifer (`coleifer`) since 2010[^1]. Its defining trait is scope: the core is one module (`peewee.py`, several thousand lines) with no required dependencies, targeting SQLite, MySQL/MariaDB, and PostgreSQL. It positions itself against SQLAlchemy not on features but on surface area — few concepts, a query API that maps almost line-for-line to SQL, and a codebase a single developer can read end to end.

The central tradeoff follows from that minimalism. Peewee gives you a precise, composable query builder and Django-like model declarations, but deliberately omits the heavy abstractions SQLAlchemy provides: there is no identity map, no unit-of-work session, no automatic dirty-tracking flush, and no autogenerating migration engine in core. You write queries that execute when iterated; you manage connections and schema changes largely yourself. For applications where that explicitness is a feature, peewee is a good fit. For large domain models with complex object graphs and change-tracking needs, the missing machinery becomes real work.

Peewee 3.0 (2018) was a ground-up rewrite of the query builder and broke meaningfully with the 2.x API[^2]. The 2026-era releases added asyncio support layered on the standard async drivers (aiosqlite, asyncpg, aiomysql)[^3]. Extensions beyond core live in the bundled `playhouse` package.

## Getting Started

```bash
pip install peewee
# SQLite works out of the box via the stdlib sqlite3 module.
pip install "peewee[postgres]"   # adds psycopg2
pip install "peewee[mysql]"      # adds pymysql
```

```python
from peewee import *
import datetime

db = SqliteDatabase('app.db')

class BaseModel(Model):
    class Meta:
        database = db

class User(BaseModel):
    username = CharField(unique=True)

class Tweet(BaseModel):
    user = ForeignKeyField(User, backref='tweets')
    message = TextField()
    created = DateTimeField(default=datetime.datetime.now)

db.connect()
db.create_tables([User, Tweet])

charlie = User.create(username='charlie')
Tweet.create(user=charlie, message='hello')

# Queries are lazy; SQL runs on iteration/execution.
recent = (Tweet
          .select(Tweet, User)
          .join(User)
          .where(User.username == 'charlie')
          .order_by(Tweet.created.desc()))
for t in recent:
    print(t.user.username, t.message)
```

## Architecture / How It Works

Model classes are built by a metaclass that scans class attributes for `Field` descriptors and records them on an internal `Meta`. At the class level, a field like `User.username` is not a value — it is a column node. Comparing it (`User.username == 'charlie'`) returns an `Expression` node rather than a boolean. Queries are trees of these `Node` objects (columns, expressions, functions via `fn`, clauses), and each `Database` subclass knows how to render that tree into dialect-specific SQL and bind parameters. This is why the query API reads like SQL: you are literally assembling an AST.

Query objects (`ModelSelect`, `ModelInsert`, `ModelUpdate`, `ModelDelete`) are lazy and composable. Each method returns a new query; nothing executes until you iterate, call `.execute()`, or coerce to a scalar. `SELECT` results are materialized into model instances by a row processor. Crucially there is **no identity map**: fetching the same row in two queries yields two distinct Python objects, and there is no session tracking their changes. You persist by calling `.save()` on an instance or by issuing an explicit `UPDATE`/`INSERT` query.

Relationships are query-backed rather than lazily proxied. A `ForeignKeyField` stores the raw id; accessing the related attribute (`tweet.user`) issues a query on first access and caches the result on the instance. The `backref` (`user.tweets`) is a query factory, not a preloaded collection. There is no automatic eager loading — you opt in with joins or `prefetch()`.

Connection state is thread-local by default. `Database.connect()`/`.close()` manage a per-thread connection; `db.atomic()` provides transactions and nested savepoints as a context manager or decorator. Everything past the core — connection pooling, schema migration, SQLite extensions, signals, Postgres-specific fields, pydantic/shortcuts helpers — lives in `playhouse`, a grab-bag of optional modules shipped in the same package.

## Production Notes

- **N+1 queries are the default failure mode.** Iterating a query and touching a foreign key or backref inside the loop issues one query per row. Use a join with `.select(Model, Related)` to populate related objects in one pass, or `prefetch()` to batch-load collections. This is the single most common peewee performance bug in production.
- **No identity map means no free deduplication or dirty-tracking.** The same DB row loaded twice is two objects; saving one does not update the other. `.save()` issues a full-row `UPDATE` unless you pass `only=[...]`. Concurrent read-modify-write needs explicit atomic `UPDATE ... SET col = col + 1` expressions, not object mutation.
- **Migrations are a known weak spot.** `playhouse.migrate` provides a `SchemaMigrator` with operations (add/drop/rename column, etc.) but no version tracking, no autogeneration, and no rollback framework. Teams typically adopt a third-party layer (`peewee-migrate`) or hand-manage migration scripts. Compared to Alembic this is a significant gap.
- **Connection pooling is not in core.** The default per-thread connection is fine for many web apps, but pooled or long-lived deployments need `playhouse.pool` (`PooledPostgresqlDatabase`, etc.), and you must ensure connections are opened/closed per request (Flask/FastAPI integration hooks exist for this).
- **SQLite tuning is manual but well-supported.** `SqliteDatabase(..., pragmas={...})` lets you set WAL mode, `foreign_keys`, cache size, and busy timeout — worth configuring explicitly, since defaults are conservative.
- **The asyncio layer is comparatively new.** The sync API is battle-tested since 2010; the async path (`pwasyncio`, `aexecute`, `asave`) is more recent and smaller in surface area — validate it against your workload rather than assuming full parity.
- **Upgrade friction is concentrated at 2.x → 3.x.** The 3.0 rewrite changed the query builder API; migrating older 2.x code is not mechanical. Within the 3.x line, releases have been comparatively stable.

## When to Use / When Not

**Use when:**
- You want an ORM you can read and reason about completely, with SQL staying visible.
- The app is small-to-medium, or query-shaped rather than object-graph-shaped.
- You value a single-file dependency and precise control over the SQL emitted.
- You are on SQLite/Postgres/MySQL and want tight, explicit connection and transaction handling.

**Avoid when:**
- You have a large domain model needing an identity map, unit-of-work, and change-tracking — SQLAlchemy's ORM is built for this.
- You need first-class autogenerated, versioned migrations out of the box.
- You are already inside Django (use its ORM) or need a database it doesn't target.
- Your team expects async as the mature default path rather than a newer addition.

## Alternatives

- sqlalchemy/sqlalchemy — use when you need a full unit-of-work, identity map, complex mappings, and Alembic-grade migrations, and can accept a larger API.
- django/django — use its built-in ORM when you are already committed to the Django stack.
- tortoise/tortoise-orm — use when you want an async-native ORM with a Django-like model API from the start.
- fastapi/sqlmodel — use when you want Pydantic models and SQLAlchemy Core unified for a FastAPI app.
- ponyorm/pony — use when you prefer expressing queries as Python generator expressions.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010-10 | First public release; SQLite/MySQL/Postgres, single module[^1]. |
| 2.0 | 2013 | Reworked query API; the long-lived 2.x line. |
| 3.0 | 2018-04 | Ground-up query-builder rewrite; breaking changes from 2.x[^2]. |
| 3.x | 2018–2026 | Steady incremental releases; playhouse extensions expanded. |
| asyncio | 2026 | asyncio support on aiosqlite/asyncpg/aiomysql[^3]. |

## References

[^1]: peewee documentation and project history, Charles Leifer. http://docs.peewee-orm.com/
[^2]: peewee changelog / release notes, "peewee 3.0" query-builder rewrite. https://github.com/coleifer/peewee/blob/master/CHANGELOG.md
[^3]: peewee asyncio documentation. https://docs.peewee-orm.com/en/latest/peewee/asyncio.html

## Tags

python, orm, sql, sqlite, postgresql, mysql, database, query-builder, asyncio, single-file
