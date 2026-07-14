# jackc/pgx

> A pure-Go PostgreSQL driver and low-level toolkit — the de facto choice when you target Postgres and only Postgres.

[GitHub repo](https://github.com/jackc/pgx) ·
[Documentation](https://pkg.go.dev/github.com/jackc/pgx/v5) ·
[License: MIT](https://github.com/jackc/pgx/blob/master/LICENSE)

## Overview

pgx is a PostgreSQL driver and toolkit for Go, written and maintained primarily
by Jack Christensen since 2013[^1]. It occupies a different niche from the older
`lib/pq`: rather than only implementing the `database/sql` contract, pgx exposes
a native interface that surfaces PostgreSQL-specific features the standard
abstraction cannot express — `LISTEN`/`NOTIFY`, the `COPY` protocol, batched
queries, binary-format encoding, and direct access to the wire protocol.

The project is two things stacked together. The **driver** is what most users
import: `pgx.Connect`, `pgxpool` for pooling, and a `stdlib` shim that registers
pgx as a `database/sql` driver for code that needs the standard interface. The
**toolkit** is the layered set of packages underneath — `pgconn`, `pgproto3`,
`pgtype` — usable on their own to build proxies, poolers, logical-replication
clients, or alternative drivers[^2]. That layering is pgx's defining tradeoff:
more power and Postgres-specific correctness than `database/sql` allows, at the
cost of coupling your data layer to an interface that is not portable.

pgx is the driver most other Go Postgres tooling builds on — `pgxpool`, `sqlc`'s
pgx output mode, `scany`, and `pgxmock` all target its interface — which in
practice makes it the ecosystem default for new Go services on PostgreSQL.

## Getting Started

```bash
go get github.com/jackc/pgx/v5
```

```go
package main

import (
	"context"
	"fmt"
	"os"

	"github.com/jackc/pgx/v5"
)

func main() {
	conn, err := pgx.Connect(context.Background(), os.Getenv("DATABASE_URL"))
	if err != nil {
		fmt.Fprintf(os.Stderr, "connect: %v\n", err)
		os.Exit(1)
	}
	defer conn.Close(context.Background())

	var name string
	var weight int64
	err = conn.QueryRow(context.Background(),
		"select name, weight from widgets where id=$1", 42).
		Scan(&name, &weight)
	if err != nil {
		fmt.Fprintf(os.Stderr, "query: %v\n", err)
		os.Exit(1)
	}
	fmt.Println(name, weight)
}
```

For servers use `pgxpool.New(ctx, url)` instead of a single `Connect` — the
pool is safe for concurrent use and acquires a connection per query.

## Architecture / How It Works

pgx is deliberately layered, each package usable independently[^2]:

- **`pgproto3`** — encodes and decodes PostgreSQL v3 frontend/backend wire
  protocol messages. Pure serialization; no I/O policy of its own.
- **`pgconn`** — the low-level connection: authentication, the frontend loop,
  `COPY`, cancellation requests, and notification delivery. This is the layer a
  proxy or custom pooler would build on.
- **`pgtype`** — the type system mapping ~70 PostgreSQL types to Go values in
  both text and binary formats. v5 rewrote this around a registry `Map` where
  each type registers a codec; custom types plug in here.
- **`pgx`** — the high-level driver most people import: `Query`, `QueryRow`,
  `Exec`, `Batch`, prepared-statement caching.
- **`pgxpool`** — the concurrency-safe connection pool (health checks,
  after-connect hooks, min/max sizing, lifetime jitter).
- **`stdlib`** — adapts a pgx connection to `database/sql` for code that needs
  the standard interface or third-party libraries that require it.

The most important internal detail for correctness is the **query exec mode**.
By default pgx uses the extended protocol with automatic prepared-statement
caching (`QueryExecModeCacheStatement`), which is fast and gives real parameter
binding. Other modes trade that off: `CacheDescribe`, `DescribeExec`, `Exec`
(extended without caching), and `Simple` (the simple query protocol, one round
trip, no server-side prepare). The mode you pick determines round trips, whether
statements are cached server-side, and — critically — whether pgx works behind a
transaction-pooling connection pooler.

v5 also added row-scanning helpers that remove most manual `rows.Scan`
boilerplate: `pgx.CollectRows` with `RowToStructByName`, `RowToStructByPos`, and
`RowTo[T]`, plus `pgx.NamedArgs` and a `QueryRewriter` hook so `@name`-style
parameters work over the positional wire protocol.

## Production Notes

**PgBouncer / transaction pooling.** The single most common footgun. In
transaction-pooling mode a pooled server connection is not stable across
statements, so pgx's default prepared-statement caching breaks with errors like
"prepared statement already exists". The fix is to disable statement caching —
set `DefaultQueryExecMode` to `QueryExecModeSimpleProtocol`, or configure the
pool to not cache — or run PgBouncer 1.21+, which added prepared-statement
support in transaction mode[^3]. Session pooling avoids the issue entirely.

**Context cancellation can close the connection.** If a query is interrupted by
context deadline or cancellation, the connection may be left in an indeterminate
protocol state; pgx's safe response is to close it rather than risk reusing a
desynchronized connection. Under load with aggressive timeouts this shows up as
churn in `pgxpool` — connections repeatedly torn down and re-dialed. Budget
timeouts realistically and prefer statement-level `statement_timeout` on the
server where you can.

**Bulk loads.** Use `CopyFrom` (the `COPY` protocol) for large inserts; it is
dramatically faster than looping `INSERT` and far faster than multi-row `INSERT`
for wide data. `Batch` reduces round trips for heterogeneous statement sets but
is not a substitute for `COPY` on bulk ingest.

**Nulls and scanning.** Scanning a SQL `NULL` into a non-pointer Go value errors.
Use pointers, `pgtype` wrapper types (`pgtype.Text`, `pgtype.Int8`), or
`sql.Null*` types. Custom domain types need a `pgtype` codec registered on the
connection's type map — a per-connection concern the pool's after-connect hook is
the right place to handle.

**Upgrades.** The v4 → v5 migration is the notable pain: the import path changed
(`/v4` → `/v5`), `pgtype` was rewritten, and custom-type integrations written
against the old `pgtype` API had to be ported. Plan it as real work, not a
find-and-replace.

## When to Use / When Not

**Use when:**
- You target PostgreSQL exclusively and want its native features (`COPY`,
  `LISTEN`/`NOTIFY`, batching, binary format, logical replication).
- You want the fastest well-maintained Go Postgres path and are fine coupling to
  a Postgres-specific API.
- You're using `sqlc`, `scany`, or `pgxpool` — they target pgx.

**Avoid when:**
- You need database portability across engines — code to `database/sql` with a
  driver you can swap (pgx's `stdlib` shim still works, but you lose the native
  API's benefits).
- You want an ORM or query builder — pgx is a driver, not that; pair it with one
  or use Bun/GORM.
- Your team already standardized on `database/sql` idioms and the Postgres-only
  features are not needed.

## Alternatives

- lib/pq — the original pure-Go Postgres `database/sql` driver, now in
  maintenance mode; use it only for legacy code, and prefer pgx for anything new.
- uptrace/bun — SQL-first ORM/query builder (runs on top of a driver like pgx);
  use it when you want mapping and query building, not just a driver.
- go-gorm/gorm — full ORM with a Postgres dialect; use when you want
  convention-heavy models and migrations over raw SQL.
- jmoiron/sqlx — thin extensions over `database/sql` (struct scanning, named
  params); use when you want portability plus a little ergonomics, not
  Postgres-native features.
- sqlc-dev/sqlc — generates type-safe Go from SQL and can emit pgx code; use it
  alongside pgx rather than instead of it.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1–v3 | 2014–2018 | Early native driver; `database/sql` shim added over time. |
| v4.0.0 | 2019 | Introduced `pgconn`, the rewritten `pgtype`, and `pgxpool`[^1]. |
| v5.0.0 | 2022 | Major rewrite: `pgtype` registry redesign, `CollectRows`/`RowToStructBy*`, `NamedArgs`, query rewriter, `QueryExecMode` API[^4]. |
| v5.x | 2023–2026 | Current stable line; new PostgreSQL type and format support, ongoing fixes. |

## References

[^1]: pgx repository and README, jackc/pgx. https://github.com/jackc/pgx
[^2]: pgx toolkit package layout (`pgconn`, `pgproto3`, `pgtype`) — package docs. https://pkg.go.dev/github.com/jackc/pgx/v5
[^3]: PgBouncer prepared-statement support in transaction mode (1.21 changelog). https://www.pgbouncer.org/changelog.html
[^4]: pgx v5 release and changelog. https://github.com/jackc/pgx/releases

## Tags

go, postgresql, database, database-driver, sql, connection-pool, wire-protocol, database-sql, backend, pgx
