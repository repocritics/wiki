# golang-migrate/migrate

> Database schema migrations for Go — a CLI and a library sharing one engine, with pluggable database and source drivers.

[GitHub repo](https://github.com/golang-migrate/migrate) ·
[License: MIT](https://github.com/golang-migrate/migrate/blob/master/LICENSE)

## Overview

migrate applies ordered, versioned schema changes to a database. It ships as
both a standalone CLI and an importable Go package
(`github.com/golang-migrate/migrate/v4`), and both run the same core loop: read
migration files from a *source* driver, apply them in order through a *database*
driver, and record progress in a version table. It is one of the most widely
used migration tools in the Go ecosystem — ~18.7k stars, ~1.6k forks, and still
actively maintained (last push July 2026)[^1].

The project is a 2018 fork of the original `mattes/migrate` by Dale Hui, after
the upstream went dormant[^2]. Its stated design philosophy is that database
drivers are "dumb": they do not correct user input, do not guess, and fail
loudly when in doubt. The engine, not the driver, owns ordering and safety
logic. This keeps the driver surface small (20+ databases are supported) at the
cost of pushing correctness decisions onto the operator.

The defining tension is minimalism versus safety nets. migrate deliberately has
no checksum validation, no squashing, no seed/data-migration DSL, and no
automatic partial-rollback. It tracks a single integer version plus a "dirty"
flag — simple and predictable, but a failed migration leaves the database in a
state you resolve by hand, the most common surprise for teams coming from Flyway
or Rails.

## Getting Started

```bash
# CLI via Homebrew (includes common drivers)
brew install golang-migrate

# or build from source, selecting drivers with build tags
go install -tags 'postgres mysql' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

```bash
# create a new migration pair (writes NNNN_create_users.up.sql / .down.sql)
migrate create -ext sql -dir db/migrations -seq create_users

# apply all pending migrations
migrate -source file://db/migrations \
        -database "postgres://localhost:5432/app?sslmode=disable" up

# roll back the last one
migrate -source file://db/migrations -database "$DATABASE_URL" down 1
```

```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

m, err := migrate.New("file://db/migrations", "postgres://localhost:5432/app?sslmode=disable")
if err != nil { /* handle */ }
if err := m.Up(); err != nil && err != migrate.ErrNoChange {
    // ErrNoChange is expected when already at latest
}
```

## Architecture / How It Works

Two driver interfaces sit on either side of the engine. **Source drivers**
(`source/file`, `source/iofs`, `source/github`, `source/aws_s3`,
`source/google_cloud_storage`, and others) enumerate migration files and stream
their contents. **Database drivers** (`database/postgres`, `database/mysql`,
`database/sqlite`, `database/clickhouse`, `database/mongodb`,
`database/spanner`, `database/cockroachdb`, `database/sqlserver`, and more)
execute a migration's body and persist the current version. The `migrate.Migrate`
type glues the two together, computing the next version and calling the driver.

Migrations are file pairs with a numeric prefix and a direction:
`1481574547_create_users.up.sql` and `..._create_users.down.sql`. Prefixes are
either Unix timestamps or sequential integers (`-seq`); either way they define a
total order. Applying "up" walks forward, "down" walks backward, and each file
runs as one unit.

State lives in a `schema_migrations` table with two columns: `version` and a
boolean `dirty`. Before running a file the engine sets `dirty = true`; on success
it clears the flag and records the new version. If the process dies or the SQL
errors mid-file, `dirty` stays set and migrate refuses to run again until you
resolve it with `migrate force <version>`. There is no checksum column — migrate
does not detect that an already-applied file was later edited, differing sharply
from Flyway's validate-on-startup behavior.

Concurrency safety is per-driver: the PostgreSQL driver takes a session-level
advisory lock so two processes cannot migrate at once, while some drivers have
weaker or no locking. Drivers are compiled in via build tags, so the binary only
carries the databases you ask for.

## Production Notes

- **The "dirty" state is the number-one footgun.** On databases without
  transactional DDL (MySQL), a migration that fails halfway leaves the schema
  partially changed *and* the version table dirty. Recovery is manual: fix the
  schema yourself, then `migrate force <version>`. PostgreSQL rolls back the
  statement but migrate still marks dirty and still needs a force.
- **Build tags decide which drivers exist.** `go install` without
  `-tags 'postgres mysql ...'` yields a CLI that fails with an opaque "unknown
  driver" error. This trips up nearly every first-time user; the Homebrew and
  Docker builds bundle common drivers.
- **Multi-statement migrations are driver-dependent.** PostgreSQL handles them
  natively; MySQL requires `x-multi-statement=true` in the URL or it errors on
  the second statement — an asymmetry that surprises teams switching engines.
- **No checksum validation.** Editing an already-applied migration is silently
  ignored; migrate only tracks the version number. Treat applied files as
  immutable by convention — the tool will not enforce it, and there is no
  built-in squashing to collapse long histories.
- **URL encoding matters.** Connection params with reserved characters (`@`,
  `/`, `#`, `%`) must be percent-encoded or the URL parse fails or misconnects[^1].
- **Library API is frozen.** v3 and v4 are documented as API-stable, so the Go
  import surface rarely breaks between releases — a real strength for embedding
  in application startup.

## When to Use / When Not

**Use when:**
- You want plain SQL migrations with an explicit up/down and a small, auditable
  tool — no DSL, no ORM coupling.
- You are a Go project and want to run migrations from application code and from
  CI/CLI using the same engine.
- You need a database migrate supports that lighter tools do not (Spanner,
  ClickHouse, CockroachDB, Neo4j, Cassandra).

**Avoid when:**
- You want guardrails: checksum validation, automatic drift detection, or
  transactional multi-file rollback. Flyway, Liquibase, or Atlas fit better.
- You want declarative schema-as-desired-state with automatic diffing rather
  than hand-written up/down pairs — use Atlas or a schema-diff tool.
- Your team will edit applied migrations and expects the tool to catch it — it
  will not.

## Alternatives

- pressly/goose — Go-native migrations that also allow Go-function (not just
  SQL) migrations; use when you need programmatic data migrations in-process.
- amacneil/dbmate — language-agnostic single-binary CLI; use when your stack
  isn't Go and you just want SQL up/down files.
- ariga/atlas — declarative, schema-as-code with diffing and linting; use when
  you want desired-state management instead of hand-ordered files.
- flyway/flyway — JVM tool with checksum validation and versioned+repeatable
  migrations; use when you want strong drift enforcement in an enterprise setting.
- rubenv/sql-migrate — lightweight Go library with embedded-asset support; use
  for smaller Go apps wanting fewer drivers and simpler embedding.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016 | Original `mattes/migrate` created by Matthias Kadenbach[^2]. |
| fork | 2018-01 | Forked to `golang-migrate/migrate` by Dale Hui after upstream stalled[^2]. |
| v3 | 2018 | First org release line; now deprecated and unsupported. |
| v4 | 2019 | Current stable line, Go-modules based, API frozen[^1]. |
| master | ongoing | New drivers and fixes land here first; last push 2026-07[^1]. |

## References

[^1]: golang-migrate/migrate README and repository metadata (stars, forks, last push, supported drivers, usage). https://github.com/golang-migrate/migrate
[^2]: Fork lineage and dual copyright (Matthias Kadenbach 2016 / Dale Hui 2018) as stated in the project LICENSE and README. https://github.com/golang-migrate/migrate/blob/master/LICENSE

## Tags

go, golang, database, database-migrations, schema-migration, sql, cli, postgres, mysql, devops, backend
