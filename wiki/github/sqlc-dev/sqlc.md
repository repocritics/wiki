# sqlc-dev/sqlc

> Compiles SQL queries into type-safe Go, Kotlin, Python, and TypeScript — you write the SQL, sqlc writes the boilerplate.

[GitHub repo](https://github.com/sqlc-dev/sqlc) ·
[Official website](https://sqlc.dev) ·
[License: MIT](https://github.com/sqlc-dev/sqlc/blob/main/LICENSE)

## Overview

sqlc is a code generator, not a runtime library. You hand it a schema (DDL) and a
set of SQL queries; it parses both against a real SQL grammar, resolves the types
of every parameter and result column, and emits idiomatic data-access code with
typed method signatures[^1]. The application you write calls those generated
methods. Nothing about sqlc is present at runtime — the generated Go talks to
`database/sql` or `pgx` directly.

This puts sqlc in a distinct category from ORMs. Where gorm or ent give you a
query builder and reflection at runtime, sqlc gives you nothing at runtime and
does all its work at `go generate` time. The payoff is that malformed SQL,
type mismatches, and typos in column names become build-time errors instead of
runtime panics. The cost is that anything sqlc cannot statically understand — most
notably dynamically-shaped queries — it will not generate for you.

The project began as `kyleconroy/sqlc` in 2019[^2] and later moved to the
`sqlc-dev` organization. PostgreSQL is the first-class engine and the one where the
analysis is deepest; MySQL and SQLite are supported but with a smaller supported
surface of SQL constructs. Go is the original and best-supported output target;
Kotlin, Python, and TypeScript are produced by separate codegen plugins[^3].

## Getting Started

```bash
# macOS / Linux via Homebrew
brew install sqlc
# or as a Go tool
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

```sql
-- schema.sql
CREATE TABLE authors (
  id   BIGSERIAL PRIMARY KEY,
  name text NOT NULL,
  bio  text
);

-- query.sql
-- name: GetAuthor :one
SELECT * FROM authors WHERE id = $1;

-- name: ListAuthors :many
SELECT * FROM authors ORDER BY name;

-- name: CreateAuthor :one
INSERT INTO authors (name, bio) VALUES ($1, $2) RETURNING *;
```

```yaml
# sqlc.yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "query.sql"
    schema: "schema.sql"
    gen:
      go:
        package: "tutorial"
        out: "tutorial"
        sql_package: "pgx/v5"
```

Running `sqlc generate` produces `models.go` (an `Author` struct) and a
`Queries` type whose methods mirror the annotated queries, e.g.
`func (q *Queries) GetAuthor(ctx context.Context, id int64) (Author, error)`.
The `:one` / `:many` / `:exec` annotation on each query determines the shape of
the generated return value.

## Architecture / How It Works

The core is a SQL parser plus a type-inference pass. sqlc does not use a hand-rolled
approximate SQL parser; for PostgreSQL it embeds the actual PostgreSQL grammar via
pg_query[^4], which means it understands the same syntax the server does. MySQL and
SQLite are handled by separate parsers, which is the structural reason those engines
support a narrower set of constructs — the parser, not sqlc's intent, is the limit.

The pipeline is: parse the schema into a catalog of tables, columns, and types →
parse each query → walk the query against the catalog to determine (a) the number
and types of input parameters and (b) the number, names, nullability, and types of
output columns → hand that resolved query to a codegen backend. Nullability
inference is why sqlc can decide between `string` and `sql.NullString` (or `*string`
under pgx) without you annotating it.

Codegen is a plugin architecture. The Go generator is built in, but Kotlin, Python,
and TypeScript are produced by WASM or process plugins loaded per the config
file[^3]. Custom generators can be written the same way — sqlc passes the resolved
query set to the plugin as a protobuf request and writes back whatever files the
plugin returns. This is what lets sqlc target languages the core team does not
maintain.

Type mapping is configurable through `overrides`: database enums become generated
Go types, custom domain types can be pointed at your own Go types, and the
`sql_package` switch (`database/sql`, `pgx/v4`, `pgx/v5`) changes both the imports
and the null-handling strategy across the entire generated package.

## Production Notes

**Dynamic queries are the defining limitation.** Because every column and parameter
must be known at generate time, queries with conditional `WHERE` clauses, variable
column lists, or optional filters cannot be expressed directly. The common
workarounds are: `sqlc.narg()` for nullable named parameters combined with
`COALESCE`/`OR $1 IS NULL` patterns; `sqlc.slice()` (pgx only) for variable-length
`IN` lists; or dropping to a hand-written query builder for the genuinely dynamic
cases. Teams that expected sqlc to replace all query construction are the ones most
often surprised here.

**sqlc does not run migrations.** It reads your schema as DDL files (or from a live
database connection) but has no migration engine of its own. Pair it with
golang-migrate, goose, or Atlas; a common failure mode is the schema files sqlc
reads drifting out of sync with what the migration tool has actually applied.

**Engine parity is uneven.** PostgreSQL analysis is the deepest and most correct.
MySQL and SQLite work for mainstream queries but hit "unsupported" errors on
constructs the Postgres path handles fine. Before committing to sqlc on MySQL or
SQLite, prototype your gnarliest real queries rather than trusting feature-parity.

**`sqlc vet` and managed databases.** `sqlc vet` lints queries against rules written
as CEL expressions, and can execute queries against a real database to catch
planner-level problems (e.g. queries that would do a full scan)[^5]. Some analysis
features connect to an ephemeral managed database, which introduces a network
dependency and account setup into the generate step if you opt in.

**Regenerate discipline.** Generated code is checked into the repo by convention.
CI should run `sqlc generate` and fail if the working tree changes, otherwise the
committed code silently diverges from the queries. Upgrading sqlc itself can change
generated output (formatting, nullability inference, import choices), so version-pin
sqlc in CI to avoid noisy diffs on unrelated PRs.

## When to Use / When Not

**Use when:**
- You want to write real SQL and get type-safe accessors without an ORM's runtime.
- Your queries are known statically and you value compile-time verification.
- You are on PostgreSQL, where sqlc's analysis is strongest.
- You want generated code you can read, check in, and step through in a debugger.

**Avoid when:**
- Your queries are heavily dynamic (user-built filters, variable column sets).
- You need runtime associations, lazy loading, or automatic migrations from your data layer.
- You are on MySQL/SQLite with exotic SQL and cannot tolerate "unsupported construct" gaps.
- Your team prefers a fluent query builder over hand-written SQL strings.

## Alternatives

- volatiletech/sqlboiler — database-first codegen ORM; use when you want models and CRUD generated by introspecting the live schema rather than from hand-written queries.
- go-gorm/gorm — full runtime ORM; use when you want associations, hooks, and auto-migration and can accept reflection overhead.
- ent/ent — code-first graph ORM; use when your domain is graph-shaped and you want typed traversals over a schema defined in Go.
- go-jet/jet — type-safe SQL builder generated from the schema; use when you want to compose queries in Go instead of writing SQL text.
- jmoiron/sqlx — thin extension over database/sql; use when you want raw SQL with light struct scanning and no codegen step at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-06 | Project created as kyleconroy/sqlc; PostgreSQL-only[^2]. |
| 1.0.0 | 2021 | First stable release. |
| 1.x | 2022–2023 | MySQL and SQLite engines; move to sqlc-dev org. |
| 1.x | 2023 | WASM/process codegen plugins; `sqlc vet` linting[^3][^5]. |
| 1.x | 2024–2026 | pgx/v5 support, ongoing engine and plugin improvements. |

## References

[^1]: sqlc README and documentation — "sqlc generates type-safe code from SQL." https://docs.sqlc.dev
[^2]: Kyle Conroy, "Introducing sqlc." https://conroy.org/introducing-sqlc
[^3]: sqlc language support and plugins. https://docs.sqlc.dev/en/latest/reference/language-support.html
[^4]: pg_query_go — embeds the PostgreSQL parser used by sqlc for Postgres analysis. https://github.com/pganalyze/pg_query_go
[^5]: sqlc vet — query linting and database-backed analysis. https://docs.sqlc.dev/en/latest/howto/vet.html

## Tags

go, sql, code-generation, codegen, postgresql, mysql, sqlite, type-safe, database, orm-alternative, developer-tools
