# sqlalchemy/alembic

> The database migration tool for SQLAlchemy — schema versioning as a directed graph of hand-editable Python scripts.

[GitHub repo](https://github.com/sqlalchemy/alembic) ·
[Official website](https://alembic.sqlalchemy.org/) ·
[License: MIT](https://opensource.org/licenses/MIT)

## Overview

Alembic is the migration tool for SQLAlchemy, written by Mike Bayer, the author
of SQLAlchemy itself[^1]. Its job is narrow and unglamorous: track the state of
a relational schema over time and emit the `ALTER` statements that move a
database from one version to the next (and, optionally, back). It has been the
de facto standard for Python schema migration outside of Django since its first
release in 2011[^1], and ships as the migration layer under Flask-Migrate,
FastAPI tutorials, and most SQLAlchemy-based stacks.

The defining design choice is that migrations are ordinary Python files linked
into a **directed acyclic graph**, not a linear log. Each script carries a
`revision` UUID-like identifier and a `down_revision` pointer; the graph can
branch, merge, and carry multiple roots and heads[^2]. This is more capable
than the integer-sequence model used by many tools, and it is also the source
of most operational friction (merge points, multiple heads, dependency edges).

The second defining choice is that autogeneration is a *starting point*, not a
source of truth. `--autogenerate` diffs your SQLAlchemy `MetaData` against a
live database and drafts a candidate script, but the maintainers are explicit
that the output must be reviewed and edited by a human[^3]. Treating it as
authoritative is the single most common way teams get burned.

## Getting Started

```bash
pip install alembic
alembic init migrations          # scaffolds env.py + versions/ + alembic.ini
```

Point `sqlalchemy.url` in `alembic.ini` at your database (or set it in
`env.py`), then wire `target_metadata` to your models' `MetaData` for
autogenerate:

```python
# migrations/env.py (excerpt)
from myapp.models import Base
target_metadata = Base.metadata
```

```bash
# draft a migration by diffing models against the DB, then REVIEW the file
alembic revision --autogenerate -m "add users.email"
alembic upgrade head             # apply everything up to the latest head
alembic downgrade -1             # step back one revision
alembic current                  # show the DB's current revision
alembic history --verbose        # print the revision graph
```

## Architecture / How It Works

A migration script exposes two functions, `upgrade()` and `downgrade()`, that
call operations on the module-level `op` proxy — `op.create_table()`,
`op.add_column()`, `op.alter_column()`, `op.execute()` for raw SQL. These are
thin builders over SQLAlchemy's own `DDLElement` constructs, so they render per
dialect without you assembling full `Table` objects.

State lives in a single-row table named `alembic_version` inside the target
database, holding the current revision id (or several rows when there are
multiple heads). Alembic never inspects timestamps or file mtimes to decide
what to run — it walks the `down_revision` graph from the stored revision to the
requested target. `env.py` is the runtime entrypoint that both `upgrade` and
`revision` execute; it builds a `MigrationContext` around either a live
`Connection` (online mode) or a buffer (offline mode).

**Autogenerate** works by reflecting the live schema, building an in-memory
`MetaData` from it, and comparing that against your declared `target_metadata`.
It detects added/removed tables, columns, indexes, and (dialect-permitting)
constraints and some type changes. It does **not** reliably detect renames (a
rename looks like a drop plus an add), server-side defaults, `CHECK`
constraints on several dialects, or column type changes that the dialect can't
introspect. Comparison behavior is tunable via `compare_type` and
`compare_server_default` hooks[^3].

**Offline mode** (`alembic upgrade head --sql`) renders migrations to a SQL
text file instead of executing them, for shops where a DBA applies DDL by hand.
Any migration that reads rows into Python at runtime (e.g. a data backfill using
`SELECT`) cannot run offline; Alembic provides `op.bulk_insert()` and
`op.execute()` as SQL-safe alternatives.

**Batch mode** exists almost entirely for SQLite, which cannot `ALTER` most
column properties. `op.batch_alter_table()` recreates the table under a temp
name, copies data, and swaps — the "move-and-copy" workflow — so a stream of
column edits collapses into one rebuild[^4]. It works on other backends too but
is rarely needed there.

## Production Notes

**Autogenerate is a draft, full stop.** Always read the generated file before
committing. Renames come out as drop-then-add (data loss), and unsupported
diffs silently produce nothing. Round-trip the migration against a scratch
copy of production before shipping.

**Set a naming convention early, or regret it.** Without an explicit
`naming_convention` on your `MetaData`, databases auto-name indexes and
constraints inconsistently, and autogenerate then can't match declared
constraints to reflected ones — producing endless spurious drop/create diffs.
Configure the convention on day one; retrofitting it means renaming live
constraints[^5].

**Multiple heads happen on every team.** Two developers branching from the same
revision both create children of it, yielding two heads. `alembic upgrade head`
then errors ambiguously; you resolve it with `alembic merge` to create a merge
revision. This is normal with the DAG model but surprises newcomers.

**No built-in locking.** Alembic does not coordinate concurrent runners. Two
app instances booting and both running `upgrade head` can race on DDL. Gate
migrations behind a single job (init container, deploy step, advisory lock) —
do not run them from every replica at startup.

**Transactional DDL is per-dialect.** PostgreSQL and SQL Server wrap migrations
in a transaction so a failure rolls back cleanly. MySQL/MariaDB issue implicit
commits on DDL, so a failed multi-step migration can leave the schema
half-applied — write those migrations to be re-runnable or split them.

**Async is supported but indirect.** With `asyncio` engines you run the
synchronous migration body via `connection.run_sync()` inside an async `env.py`;
the async templates from `alembic init -t async` handle the wiring[^6].

**Data migrations are your problem.** Alembic versions schema, not data. Large
backfills belong in `op.execute()` with batched SQL or a separate job, not in a
loop that pulls rows into Python — that neither scales nor runs in offline mode.

## When to Use / When Not

**Use when:**
- Your app already uses SQLAlchemy (Core or ORM) — Alembic is the native fit.
- You need branch/merge migration history across a team or multiple deploy lines.
- You need both direct-apply and DBA-friendly offline SQL output from one source.
- You target SQLite in dev and need `ALTER`-heavy changes via batch mode.

**Avoid when:**
- You use Django — its built-in migrations are tighter integrated; don't bolt on Alembic.
- You want a language-agnostic, SQL-file-first tool your non-Python services share.
- You want fully automatic, review-free migrations — Alembic deliberately isn't that.

## Alternatives

- django/django — use Django's built-in migrations instead when your ORM is the Django ORM, not SQLAlchemy.
- golang-migrate/migrate — use when you want plain versioned `.sql` files driven from a CLI across many languages.
- flyway/flyway — use in JVM/enterprise shops that want SQL-first migrations with commercial support.
- liquibase/liquibase — use when you need a database-agnostic changelog (XML/YAML/SQL) and rollback tooling across many DB vendors.
- pressly/goose — use for lightweight Go projects wanting SQL or Go migrations without a heavy framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2011-11 | Initial release by Mike Bayer as SQLAlchemy's migration tool[^1]. |
| 0.7 | 2014-12 | Batch (move-and-copy) migrations for SQLite[^4]. |
| 0.8 | 2015-05 | Naming conventions integrated with autogenerate constraint matching. |
| 1.0 | 2018-07 | API stabilization; drops legacy Python support. |
| 1.5 | 2021-01 | Post-write hooks; async migration templates groundwork. |
| 1.7 | 2021-08 | Async engine support via `run_sync` templates[^6]. |
| 1.13 | 2023-12 | Ongoing SQLAlchemy 2.x alignment and typing improvements. |

## References

[^1]: Alembic front matter, "written by the author of SQLAlchemy." SQLAlchemy project overview. https://www.sqlalchemy.org/
[^2]: Alembic docs, "Working with Branches" — revision DAG, heads, merges. https://alembic.sqlalchemy.org/en/latest/branches.html
[^3]: Alembic docs, "Auto Generating Migrations" — what is and isn't detected; review requirement. https://alembic.sqlalchemy.org/en/latest/autogenerate.html
[^4]: Alembic docs, "Running 'Batch' Migrations for SQLite and Other Databases." https://alembic.sqlalchemy.org/en/latest/batch.html
[^5]: Alembic docs, "The Importance of Naming Constraints." https://alembic.sqlalchemy.org/en/latest/naming.html
[^6]: Alembic docs, "Using Asyncio with Alembic." https://alembic.sqlalchemy.org/en/latest/cookbook.html#using-asyncio-with-alembic

## Tags

python, database, migrations, sqlalchemy, orm, schema-management, sql, ddl, devops, cli
