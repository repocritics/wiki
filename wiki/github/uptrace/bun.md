# uptrace/bun

> SQL-first Go ORM built on `database/sql` — a query builder that maps to SQL rather than hiding it, with pluggable dialects for PostgreSQL, MySQL, MSSQL, SQLite, and Oracle.

[GitHub repo](https://github.com/uptrace/bun) ·
[Official website](https://bun.uptrace.dev) ·
[License: BSD-2-Clause](https://github.com/uptrace/bun/blob/master/LICENSE)

## Overview

Bun is a Go ORM that positions itself as "SQL-first": instead of abstracting SQL away behind an object graph, it exposes a fluent builder (`db.NewSelect()...`, `db.NewInsert()...`) whose method chain reads close to the SQL it emits. It is the successor to `go-pg` — a PostgreSQL-only ORM by the same primary author (Vladimir Mihailenco) — generalized to run on top of the standard `database/sql` package so it can target multiple databases through swappable dialects[^1]. `go-pg` is in maintenance mode and its README points new projects at Bun.

The central tradeoff is deliberate: Bun does very little to protect you from SQL. Column names, expressions, and `Where` fragments are passed as strings (`Where("total_sales > ?", n)`), so a typo in a column reference is a runtime error, not a compile error. In exchange you get a thin, predictable layer over `database/sql`, no code generation step, and near-total control over the emitted query. This puts it between GORM (heavier, more magic, more abstraction) and code-generation tools like `sqlc` or `ent` (compile-time-checked, but a build step and a different mental model).

Bun is maintained by Uptrace, an open-source APM/observability vendor; the project ships first-class OpenTelemetry instrumentation via a query hook, and the README funnels toward Uptrace's hosted product. The library itself is BSD-2-Clause and does not require any Uptrace service to run.

## Getting Started

```bash
go get github.com/uptrace/bun
go get github.com/uptrace/bun/dialect/sqlitedialect
go get github.com/uptrace/bun/driver/sqliteshim
```

```go
package main

import (
	"context"
	"database/sql"
	"fmt"

	"github.com/uptrace/bun"
	"github.com/uptrace/bun/dialect/sqlitedialect"
	"github.com/uptrace/bun/driver/sqliteshim"
)

type User struct {
	ID   int64  `bun:",pk,autoincrement"`
	Name string `bun:",notnull"`
}

func main() {
	ctx := context.Background()

	sqldb, _ := sql.Open(sqliteshim.ShimName, "file::memory:?cache=shared")
	db := bun.NewDB(sqldb, sqlitedialect.New())

	db.NewCreateTable().Model((*User)(nil)).Exec(ctx)

	user := &User{Name: "Ada"}
	db.NewInsert().Model(user).Exec(ctx)

	err := db.NewSelect().Model(user).Where("id = ?", user.ID).Scan(ctx)
	fmt.Printf("%+v %v\n", user, err)
}
```

Note the `Model((*User)(nil))` idiom: Bun uses a typed nil pointer to pass the model *type* (not an instance) so it can read struct tags for schema and column metadata.

## Architecture / How It Works

Bun wraps `*sql.DB` rather than replacing it. `bun.NewDB(sqldb, dialect)` takes an already-opened `database/sql` handle plus a **dialect** that knows how to render SQL for a given backend. This has two consequences: connection pooling, drivers, and context handling are whatever `database/sql` gives you, and the ORM's job is confined to building query strings and scanning results.

- **Dialects.** `pgdialect`, `mysqldialect`, `sqlitedialect`, `mssqldialect`, `oracledialect` handle SQL syntax differences (placeholders, quoting, `RETURNING`, upsert syntax). Swapping dialects is how "database-agnostic" code works — though non-trivial queries still leak dialect assumptions.
- **Drivers.** Bun ships its own pure-Go PostgreSQL driver (`pgdriver`) and a SQLite shim (`sqliteshim`, wrapping `modernc.org/sqlite` or `mattn/go-sqlite3`). For MySQL, MSSQL, and Oracle you bring the community driver. You can also pair `pgdialect` with `jackc/pgx`'s `stdlib` adapter instead of `pgdriver`.
- **Model mapping.** Structs are annotated with `bun:"..."` tags: primary keys, `notnull`, `autoincrement`, `soft_delete`, custom column names, and relations.
- **Relations.** `has-one`, `belongs-to`, `has-many`, and `many-to-many` (`m2m`) are declared via tags and loaded with `.Relation("Posts")`. `belongs-to`/`has-one` can be rendered as JOINs; `has-many`/`m2m` are loaded as separate follow-up queries (an `IN (...)` per relation), which sidesteps N+1 loops but issues multiple round trips.
- **Query hooks.** `db.AddQueryHook(...)` intercepts every query — this is the extension point for `bundebug` (query logging) and `bunotel` (OpenTelemetry spans and metrics). Hooks see the rendered query and timing.
- **Migrations & fixtures.** `bun/migrate` supports both Go-function migrations and `.up.sql`/`.down.sql` files, tracked in a migrations table. `dbfixture` loads YAML seed data for tests.

Because the builder emits SQL directly, the "ORM" never hides the query: `bundebug` or `q.String()` shows exactly what runs, and dropping to raw SQL (`db.QueryContext`, `db.NewRaw(...)`) is a first-class escape hatch rather than a defeat.

## Production Notes

- **No compile-time query safety.** Columns and expressions are strings. A renamed column or a typo in `Where`/`ColumnExpr` fails at runtime, and only on the code path that executes it. Teams that want compile-checked queries reach for `sqlc`, `ent`, or `go-jet` instead — that is the primary reason to *not* pick Bun.
- **`pgdriver` vs `pgx`.** Bun's built-in `pgdriver` is pure Go and convenient, but it is less feature-rich and less battle-tested than `jackc/pgx`. Under heavy PostgreSQL load many users run `pgdialect` on top of `pgx`'s `stdlib` driver for its connection handling, prepared-statement, and protocol features. Evaluate before committing to `pgdriver` in high-throughput services.
- **Connection pool is yours.** Bun does not manage pooling. `SetMaxOpenConns`, `SetMaxIdleConns`, and `SetConnMaxLifetime` are set on the underlying `*sql.DB`; forgetting to tune them is the usual cause of connection exhaustion or idle-connection churn.
- **Bulk operation parameter limits.** Bulk insert/update generate one large statement with a parameter per value. PostgreSQL caps bind parameters at 65535 per statement; large batches must be chunked or they fail at the protocol level.
- **Soft deletes change query semantics.** A `soft_delete` field makes `Delete` issue an `UPDATE ... SET deleted_at`, and `Select` implicitly filters deleted rows. Restoring or hard-deleting requires `WhereAllWithDeleted()` / `ForceDelete()`; unaware code silently sees a filtered view of the table.
- **Two migration styles, one table.** Mixing Go and SQL migrations works but the ordering is by filename/registration order — keep a single, consistent naming convention or migrations apply out of intended sequence.
- **Release cadence.** Development is steady but not fast-moving: the API has been stable across the v1.2.x line, and the project is on a slower, maintenance-heavy rhythm rather than rapid feature churn — a plus for stability, a signal to check issue activity if you need a specific new capability.

## When to Use / When Not

**Use when:**
- You want SQL you can read and predict, with an ORM that stays out of the query.
- You target more than one SQL database from one codebase and accept dialect-aware querying.
- You want migrations, relations, soft deletes, fixtures, and OpenTelemetry without a code-generation build step.
- You're migrating off `go-pg` — Bun is its intended successor.

**Avoid when:**
- You want compile-time-verified queries and typed result structs generated from schema (`sqlc`, `ent`, `go-jet`).
- You want a batteries-included, convention-heavy ORM with auto-migration and hooks everywhere (GORM fits that mold better).
- Your team dislikes string-based column references and would rather trade flexibility for static guarantees.

## Alternatives

- go-gorm/gorm — use instead when you want the most popular Go ORM with heavier abstraction, auto-migration, and a large plugin ecosystem, and you don't mind more magic.
- sqlc-dev/sqlc — use instead when you'd rather write plain SQL and generate type-safe Go from it at build time, with no runtime query builder.
- ent/ent — use instead when you want a graph/schema-as-code model with generated, compile-checked query APIs (originally from Facebook).
- go-jet/jet — use instead when you want a fully type-safe SQL builder generated from your live database schema.
- uptrace/go-pg — Bun's PostgreSQL-only predecessor; in maintenance mode, use Bun for new PostgreSQL projects.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2021-05 | Repo created; generalization of `go-pg` onto `database/sql`[^1]. |
| v1.0.0 | 2021-09-01 | First stable release, multi-dialect API. |
| v1.1.0 | 2022-02-28 | Relations, migration, and API refinements. |
| v1.2.0 | 2024-04-02 | Continued v1.2.x line; API stabilized. |
| v1.2.18 | 2026-02-28 | Latest tagged release as of this writing. |

## References

[^1]: Bun documentation, "Golang ORM" and project background (successor to go-pg). https://bun.uptrace.dev/guide/golang-orm.html
[^2]: Bun README, database/driver/dialect support matrix. https://github.com/uptrace/bun#readme
[^3]: `uptrace/go-pg` — PostgreSQL client and ORM, now in maintenance mode. https://github.com/uptrace/go-pg

## Tags

go, golang, orm, sql, database, postgresql, mysql, sqlite, mssql, query-builder, database-sql, migrations
