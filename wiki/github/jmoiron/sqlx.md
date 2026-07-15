# jmoiron/sqlx

> A thin set of extensions on Go's `database/sql` — struct scanning and named parameters, not an ORM.

[GitHub repo](https://github.com/jmoiron/sqlx) ·
[Official docs](http://jmoiron.github.io/sqlx/) ·
[License: MIT](https://github.com/jmoiron/sqlx/blob/master/LICENSE)

## Overview

sqlx is a library by Jason Moiron that layers convenience methods on top of the
Go standard library's `database/sql` package[^1]. Its central design decision is
non-invasiveness: `sqlx.DB`, `sqlx.Tx`, `sqlx.Stmt`, and friends wrap the
standard types and expose a strict superset of their interfaces. Existing
`database/sql` code compiles against sqlx with a type change and nothing else,
which is why it became the default "just enough" data-access layer in the Go
ecosystem for teams that did not want a full ORM.

The value it adds is narrow and specific: reflection-based scanning of query
rows into structs, maps, and slices (`Get`, `Select`, `StructScan`); named
parameters (`:name` binding via `NamedExec` / `NamedQuery`); and query helpers
like `sqlx.In` for expanding slice arguments into `IN (?, ?, ?)` clauses. It does
not manage schema, relationships, migrations, or query building. You still write
SQL by hand, still open a driver-specific connection, and still handle NULLs
with `sql.Null*` types or pointers.

The defining tension is exactly this minimalism. sqlx removes the most tedious
`rows.Scan` boilerplate without hiding SQL, but it stops well short of an ORM, so
anything relational (joins into nested structs, eager loading, dynamic query
construction) remains your problem. Teams either love it for staying out of the
way or outgrow it and reach for a code generator or ORM.

## Getting Started

```bash
go get github.com/jmoiron/sqlx
```

```go
package main

import (
    "log"

    _ "github.com/lib/pq"
    "github.com/jmoiron/sqlx"
)

type Person struct {
    FirstName string `db:"first_name"`
    LastName  string `db:"last_name"`
    Email     string // maps to column "email" (lowercased field name)
}

func main() {
    // Connect = Open + Ping; use sqlx.Open for lazy sql.Open semantics.
    db, err := sqlx.Connect("postgres", "user=foo dbname=bar sslmode=disable")
    if err != nil {
        log.Fatalln(err)
    }

    // Select scans every row into a slice; Get scans exactly one row.
    var people []Person
    if err := db.Select(&people, "SELECT * FROM person ORDER BY first_name"); err != nil {
        log.Fatalln(err)
    }

    var jason Person
    if err := db.Get(&jason, "SELECT * FROM person WHERE first_name=$1", "Jason"); err != nil {
        log.Fatalln(err)
    }

    // Named parameters bind from a struct or map[string]interface{}.
    _, err = db.NamedExec(
        `INSERT INTO person (first_name, last_name, email)
         VALUES (:first_name, :last_name, :email)`, &jason)
    if err != nil {
        log.Fatalln(err)
    }
}
```

Context-aware variants (`GetContext`, `SelectContext`, `NamedExecContext`, etc.)
exist for every method and should be preferred in servers.

## Architecture / How It Works

The core is the `reflectx` mapper. On first use for a given struct type, sqlx
walks the type with reflection, honoring `db` struct tags and embedded structs,
and builds a field-name-to-index map. That map is cached per type, so the
reflection cost is paid once, not per row. `StructScan` then uses `Rows.Columns()`
to line column names up against the cached map and assigns via reflection.

Named-query support is a pre-processor. sqlx parses `:name` tokens out of the SQL
string, rewrites them to the driver's positional bindvar (`$1` for Postgres, `?`
for MySQL/SQLite, `:N` for Oracle-style), and reorders the argument list to
match. The driver's bindvar style is inferred from the `driverName` passed to
`Open`/`Connect`; `sqlx.BindDriver` lets you register a bind type for a driver
sqlx doesn't recognize (introduced in 1.3.0)[^2]. `sqlx.In` performs a similar
rewrite, expanding a slice argument into the right number of placeholders, and
`Rebind` translates `?` placeholders to a target driver's style.

Everything is built by embedding the standard types. `sqlx.DB` embeds `*sql.DB`;
the extension methods are additions, not overrides. This keeps sqlx honest about
what it is — there is no connection pool of its own, no statement cache beyond
what `database/sql` provides, and no query dialect abstraction. The coupling to
`database/sql` is total and deliberate: sqlx is only as capable, and only as
buggy, as the standard library and the underlying driver it wraps.

## Production Notes

**It is stable but slow-moving.** The last tagged release, 1.3.5, predates the
repository's most recent commit activity by a wide margin, and there are several
hundred open issues against a small maintainer surface. In practice sqlx is
"finished" software: the API is settled and widely depended upon, but do not
expect fixes or new features on any schedule. Pin the version and read the diff
before upgrading.

**`SELECT *` into a struct is a footgun on missing fields.** By default,
`StructScan` errors if a returned column has no matching destination field. This
protects against typos but breaks the moment someone adds a column to the table
or the query. `db.Unsafe()` returns a handle that silently ignores unmapped
columns — convenient, but it also hides genuine mapping mistakes, so use it
deliberately rather than reflexively.

**Ambiguous columns are unresolved.** Queries like `SELECT a.id, b.id FROM foo a
JOIN foo b ...` produce duplicate column names, and `Columns()` does not qualify
them, so struct/map destinations become ambiguous[^1]. The maintainers' guidance
is to alias columns with `AS`, scan manually with `rows.Scan`, or use `SliceScan`.
There is no automatic join-to-nested-struct support; sqlx scans flat rows only.

**NULLs are your responsibility.** A `NULL` column scanned into a plain `string`
or `int` fails at runtime. You must use `sql.NullString` / `sql.NullInt64` / a
pointer type for every nullable column, which leaks database nullability into
your domain structs — a common source of friction.

**Named parameters bind values, not identifiers.** `:name` protects value
arguments against injection, but table and column names cannot be parameterized;
building those from user input is unsafe regardless of sqlx. There is no query
builder, so dynamic `WHERE` clauses are string concatenation you write yourself.

**Reflection allocates.** `Select` into a large slice allocates per row and per
field; for hot read paths over big result sets, manual `rows.Next()` + `Scan`, or
a code-generation approach, will measurably reduce garbage. For typical CRUD
workloads the difference is irrelevant.

## When to Use / When Not

**Use when:**
- You want to keep writing raw SQL but are tired of `rows.Scan` boilerplate.
- You have an existing `database/sql` codebase and want an incremental upgrade.
- You value a tiny, stable dependency with no code generation step.
- Your access patterns are mostly single-table CRUD and simple queries.

**Avoid when:**
- You want compile-time-checked queries — a generator catches errors sqlx finds
  only at runtime.
- You need associations, eager loading, migrations, or a query builder.
- You build heavily dynamic SQL and want structured composition rather than
  string concatenation.
- You require an actively maintained library with regular releases.

## Alternatives

- sqlc-dev/sqlc — generates type-safe Go from your SQL and schema; use when you
  want queries checked at compile time instead of at runtime.
- go-gorm/gorm — full ORM with associations, hooks, and migrations; use when you
  want the database abstracted away rather than exposed.
- ent/ent — schema-as-code with generated, graph-aware clients; use when models
  and relationships are complex and codegen is acceptable.
- Masterminds/squirrel — a fluent SQL builder; use when you mainly need to
  compose dynamic queries and can scan results yourself.
- uptrace/bun — SQL-first ORM-lite with a query builder over `database/sql`; use
  when you want more than sqlx but less than GORM.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2013-01-28 | Extensions on `database/sql` begin[^1]. |
| pre-modules | 2013–2019 | Grew as the de facto Go data-access helper; unversioned imports. |
| 1.2.0 | 2019 | Go modules support; `go.mod` published. |
| 1.3.0 | 2021 | `Connx`, `BindDriver`, batch inserts via `[]map[string]interface{}`, `sqlx.In` perf[^2]. |
| 1.3.5 | 2022 | Latest tagged release as of this writing; bugfixes over the 1.3 line. |

## References

[^1]: sqlx README and issues section, jmoiron/sqlx. https://github.com/jmoiron/sqlx
[^2]: sqlx README, "Recent Changes" (1.3.0). https://github.com/jmoiron/sqlx#recent-changes

## Tags

go, golang, database, sql, database-sql, orm-alternative, struct-scanning, named-parameters, data-access, postgres, mysql, sqlite
