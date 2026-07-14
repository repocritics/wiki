# go-gorm/gorm

> The most widely used ORM for Go — convention-driven, reflection-heavy, and built around a pluggable callback pipeline.

[GitHub repo](https://github.com/go-gorm/gorm) ·
[Official website](https://gorm.io) ·
[License: MIT](https://github.com/go-gorm/gorm/blob/master/LICENSE)

## Overview

GORM is a full-featured object-relational mapper for Go, authored by Jinzhu Zhang and first released in 2013[^1]. It maps Go structs to SQL tables by convention (an `ID` field becomes the primary key, `CreatedAt`/`UpdatedAt`/`DeletedAt` are managed automatically), and covers associations, eager loading, transactions, hooks, and schema auto-migration behind a chainable API. It sits on top of the standard `database/sql` package and delegates dialect-specific SQL to swappable driver packages (`gorm.io/driver/mysql`, `postgres`, `sqlite`, `sqlserver`, `clickhouse`).

The current line is **v2** (`gorm.io/gorm`), a 2020 ground-up rewrite of the original `github.com/jinzhu/gorm` v1[^2]. v2 replaced the internal engine with a callback-driven statement builder, added context and prepared-statement support, and broke the old import path and much of the API. Projects still on v1 are on an unmaintained codebase and the upgrade is non-trivial.

The defining tension is convention and ergonomics versus explicitness and predictability. GORM optimizes for terse, readable data-access code, and pays for it with heavy runtime reflection, implicit behavior (automatic soft-delete filters, magic timestamp fields, silent condition accumulation), and a performance floor well above hand-written SQL. Teams that value transparent, auditable queries increasingly reach for query builders or codegen tools instead; teams that want to move fast on CRUD-shaped domains stay. As of 2026 it remains the default answer to "which ORM for Go," with ~39.9k stars and active but measured maintenance (last push 2026-06).

## Getting Started

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/sqlite
```

```go
package main

import (
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

// gorm.Model embeds ID, CreatedAt, UpdatedAt, DeletedAt.
type Product struct {
	gorm.Model
	Code  string
	Price uint
}

func main() {
	db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}

	db.AutoMigrate(&Product{})

	db.Create(&Product{Code: "D42", Price: 100})

	var product Product
	db.First(&product, "code = ?", "D42") // WHERE code = 'D42' ORDER BY id LIMIT 1

	db.Model(&product).Update("Price", 200)

	if err := db.Error; err != nil { // errors surface on the *gorm.DB, not returned
		panic(err)
	}
}
```

## Architecture / How It Works

GORM's core is a **callback pipeline**, not a query compiler. Each of the five operation kinds — Create, Query, Update, Delete, Row/Raw — has an ordered list of registered callback functions. A finisher method (`First`, `Find`, `Create`, `Save`, …) builds a `*gorm.Statement`, runs it through the callbacks for that operation, and each callback mutates the statement's `Clause` map or executes SQL. Plugins are just code that registers additional callbacks into this pipeline; the official Prometheus, database-resolver (read/write splitting), and optimistic-locking plugins all work this way. This is the extensibility mechanism and the reason behavior can be hard to trace: what runs on a `Create` depends on what has been registered.

Chain methods (`Where`, `Select`, `Joins`, `Preload`, `Order`) do not execute — they clone the session and accumulate `clause.Expression` values. SQL is only generated when a finisher runs, at which point the `Dialector` for the active driver renders dialect-specific syntax (quoting, `LIMIT`/`OFFSET`, upsert clauses, returning). Struct metadata (fields, tags, associations) is parsed once via reflection and cached in a `schema.Schema`, so the reflection cost is mostly first-use, but per-row scanning still goes through reflection.

Two design choices dominate day-to-day use. First, **soft deletes**: any model with a `gorm.DeletedAt` field (which `gorm.Model` includes) has `WHERE deleted_at IS NULL` injected into every query automatically, and `Delete` becomes an `UPDATE`. Second, **session/chain safety**: a `*gorm.DB` carries accumulated conditions, so reusing one that already has a `Where` applied silently leaks that condition into later queries. The intended pattern is to start each logical query from the base handle or an explicit `db.Session(&gorm.Session{})`.

## Production Notes

**AutoMigrate is not a migration tool.** `AutoMigrate` is additive and best-effort: it creates missing tables, columns, and indexes, but it will not drop columns, will not reliably alter types, and offers no down-migrations or versioning[^3]. It is fine for local development and prototyping. For production schema evolution, pair GORM with a real migration tool (golang-migrate/migrate or ariga/atlas) and treat schema as a separate, reviewed artifact.

**N+1 via Preload.** `Preload` issues a separate query per association rather than a join, so nested and sibling preloads multiply query count. `Joins` avoids the extra round-trips for has-one/belongs-to but does not populate slice associations. Enable the slow-query logger (`logger.Config{SlowThreshold}`) and watch query counts under realistic data; unbounded `Preload` on list endpoints is the most common performance regression.

**Error handling is easy to miss.** Errors do not come back from chain calls — they accumulate on `db.Error`, and "no rows" surfaces as `gorm.ErrRecordNotFound` only from single-record finishers (`First`, `Take`, `Last`), not from `Find` into a slice (which returns an empty slice and nil error). Code that forgets to check `db.Error` or that expects `Find` to error on empty results is a recurring bug source.

**Reflection overhead.** Every create and scan traverses cached struct metadata via reflection. For hot paths this is measurably slower than `database/sql` with hand-written scans or codegen ORMs; when a query dominates a latency budget, drop to `db.Raw(...).Scan(...)` or bypass GORM entirely. Enable **prepared-statement mode** (`PrepareStmt: true`) to cache statements across calls.

**Soft-delete footguns.** Because `deleted_at IS NULL` is implicit, unique indexes collide with soft-deleted rows (a "deleted" record still occupies the unique value), `COUNT`/aggregate queries silently exclude soft-deleted rows, and forgetting the behavior exists leads to confusion about "missing" data. Use `Unscoped()` to see or hard-delete them, and consider partial unique indexes at the database level.

**Upgrade pain.** v1 (`jinzhu/gorm`) to v2 (`gorm.io/gorm`) is a rewrite, not a bump: import paths, error handling, `AutoMigrate` semantics, and association APIs all changed. v1 should be considered end-of-life. Within v2, driver packages are versioned separately from core, so a `gorm.io/gorm` upgrade may require matching driver updates.

## When to Use / When Not

**Use when:**
- Your domain is CRUD-shaped and you want to move fast on standard create/read/update/delete over relational data.
- You want associations, hooks, soft deletes, and transactions without hand-writing the plumbing.
- You need to target multiple SQL databases from one codebase with minimal dialect-specific code.
- Your team already knows GORM conventions and values write-time ergonomics over query transparency.

**Avoid when:**
- Query performance and predictability are first-order requirements — reflection overhead and implicit behavior work against you.
- You want compile-time-checked SQL: a codegen tool that generates typed methods from `.sql` files fits better.
- Your workload is analytical / read-heavy with complex joins where you'd end up writing raw SQL anyway.
- You value explicit, auditable data access and dislike magic (auto soft-delete filters, condition leakage, silent empty-result handling).

## Alternatives

- sqlc/sqlc — generates type-safe Go from hand-written SQL; use when you want real SQL with compile-time checking and no reflection.
- ent/ent — Meta's graph/entity framework with codegen and a typed builder; use when your model is graph-shaped and you want static guarantees.
- uptrace/bun — lightweight SQL-first ORM over `database/sql`; use when you want ORM conveniences but explicit queries.
- jmoiron/sqlx — thin extension of `database/sql` for struct scanning; use when you want almost no abstraction over raw SQL.
- Masterminds/squirrel — a SQL builder only (no mapping); use when you just want to compose queries programmatically.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1 (jinzhu/gorm) | 2013-10 | Initial release; the original callback-based ORM[^1]. |
| v2.0 (gorm.io/gorm) | 2020-08 | Full rewrite: new statement/callback engine, context, prepared statements, new import path[^2]. |
| gorm.io/gen | 2021-2022 | Companion code generator for type-safe DAO/query APIs. |
| v1 EOL (de facto) | ~2020+ | `jinzhu/gorm` frozen after v2; treated as unmaintained. |

## References

[^1]: go-gorm/gorm repository and history — original release 2013. https://github.com/go-gorm/gorm
[^2]: "GORM V2 Release Note" — GORM documentation. https://gorm.io/docs/v2_release_note.html
[^3]: "Migration — Auto Migration" (limitations of AutoMigrate) — GORM documentation. https://gorm.io/docs/migration.html

## Tags

go, golang, orm, database, sql, active-record, database-sql, postgres, mysql, sqlite, data-access
