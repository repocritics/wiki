# pressly/goose

> A database migration tool for Go — SQL files or Go functions, usable as both a CLI and an embedded library.

[GitHub repo](https://github.com/pressly/goose) ·
[Official website](https://pressly.github.io/goose/) ·
[License: MIT](https://github.com/pressly/goose/blob/main/LICENSE)

## Overview

goose is a schema-migration tool for Go projects. It applies ordered, versioned changes to a database — each migration is either a `.sql` file with up/down blocks or a Go function registered at compile time — and records which have run in a `goose_db_version` bookkeeping table. It ships as a single CLI binary (`go install .../cmd/goose`) and as a Go package you import into your own application, which is the feature that distinguishes it from language-agnostic tools: your migrations can call arbitrary Go, share your app's driver, and be embedded into your binary via `embed.FS`.

The project began as Liam Staskawicz's `goose` in 2012 and was adopted and rewritten under the pressly organization, whose repository dates to 2016[^1]. The current major version is `v3`, imported as `github.com/pressly/goose/v3`; the module path carries the version suffix, so mixing v2 and v3 import paths in one build is a common early mistake. It is broadly used in the Go server ecosystem and is actively maintained, with releases landing regularly as of 2026[^2].

goose's defining tension is that it is deliberately simple and unopinionated — no schema DSL, no diffing engine, just "run these files in order" — which makes it easy to reason about but pushes correctness (especially the correctness of `down` migrations) entirely onto the author. It sits between hand-rolled SQL scripts and declarative schema-as-code tools like Atlas.

## Getting Started

```shell
go install github.com/pressly/goose/v3/cmd/goose@latest   # CLI
# or: brew install goose
```

Create and apply a SQL migration:

```shell
export GOOSE_DRIVER=postgres
export GOOSE_DBSTRING="postgres://user:pass@localhost:5432/app?sslmode=disable"
export GOOSE_MIGRATION_DIR=./migrations

goose create add_users sql        # writes 20260715_add_users.sql
goose up                          # apply all pending
goose status                      # show applied / pending
goose down                        # roll back one
```

A migration file uses annotation comments to separate directions:

```sql
-- +goose Up
CREATE TABLE users (id serial PRIMARY KEY, email text NOT NULL);

-- +goose Down
DROP TABLE users;
```

As a library, with migrations embedded into the binary:

```go
//go:embed migrations/*.sql
var embedMigrations embed.FS

func migrate(db *sql.DB) error {
    goose.SetBaseFS(embedMigrations)
    if err := goose.SetDialect("postgres"); err != nil {
        return err
    }
    return goose.Up(db, "migrations")
}
```

## Architecture / How It Works

goose tracks state in a single table (default `goose_db_version`, overridable with `-table`) holding a row per applied migration: version id, timestamp, and an `is_applied` flag. "Current version" is the highest applied id. There is no content hashing or checksum of migration files — goose trusts filenames and the version table, so editing an already-applied migration is silently ignored, and renaming one corrupts history.

**SQL migrations** are parsed by a small annotation scanner. `-- +goose Up` / `-- +goose Down` mark the two directions; statements are split on semicolons. Because that splitter is naive, any statement containing internal semicolons (PL/pgSQL functions, triggers, `BEGIN ... END` blocks) must be wrapped in `-- +goose StatementBegin` / `-- +goose StatementEnd`. `-- +goose NO TRANSACTION` opts a file out of the automatic transaction wrapper (required for `CREATE DATABASE`, Postgres `CREATE INDEX CONCURRENTLY`, etc.). `-- +goose ENVSUB ON/OFF` toggles environment-variable interpolation, off by default for backward compatibility.

**Go migrations** are registered by an `init()` function calling `goose.AddMigration(up, down)` (or the `Context` / `NoTx` variants). Because registration is a global side effect, a Go migration only exists if its package is imported into the build — hence you must compile your own goose binary or drive it through the library.

Dialect support (Postgres, MySQL/MariaDB, SQLite, MSSQL, ClickHouse, Spanner, YDB, Vertica, TiDB, Redshift, StarRocks, Turso/libSQL, and more) is pluggable. Drivers are compiled in via build tags, so the default binary bundles every driver; `go build -tags='no_mysql no_clickhouse ...'` produces a lean build with only what you need.

The legacy top-level API (`goose.Up`, `goose.AddMigration`) relies on a **global migration registry and global dialect state**, which is awkward for tests and multi-tenant use. Newer code should prefer the **`goose.NewProvider`** API, which encapsulates the store, filesystem, and registered migrations in an explicit value with no package-level globals[^3].

## Production Notes

- **Down migrations are the biggest footgun.** goose runs whatever `-- +goose Down` you wrote; it has no way to verify a rollback actually reverses the up. Down blocks are routinely under-tested and drift out of sync. Many teams treat migrations as forward-only in production and never run `down` against live data. `down-to 0` will unwind *everything* — there is no confirmation prompt.
- **No file checksums.** Unlike Flyway/Liquibase, goose does not detect that an applied migration's contents changed. This keeps it simple but removes a safety net; enforce immutability of merged migrations by team convention or CI.
- **Out-of-order migrations error by default.** If two branches add migrations `005` and `006` and land in the other order, goose refuses to apply the "missing" earlier one unless you pass `-allow-missing` / `WithAllowMissing()`. The maintainers instead recommend **hybrid versioning**: author with timestamps, then run `goose fix` in CI to renumber sequentially before release[^4].
- **Transaction semantics are database-dependent.** Postgres has transactional DDL, so a failed multi-statement migration rolls back cleanly. MySQL/MariaDB do **not** — a mid-migration failure leaves partially applied schema that goose cannot undo. Split risky MySQL changes into separate migrations.
- **MySQL driver flags.** The `mysql` driver requires `parseTime=true`, and multi-statement SQL files require `multiStatements=true` in the DSN or they will fail.
- **Schema / table placement.** Under a non-`public` Postgres schema you must qualify the version table (`-table='myschema.goose_db_version'`), or goose will look in the wrong place and re-run everything.
- **Embedded FS is read-only.** `goose.SetBaseFS(embed.FS)` works for `up`/`down`/`status` but not `create`/`fix`, which still write to the OS filesystem.
- **v2 → v3 upgrade** changed the import path to `.../v3` and reorganized dialects; the CLI flags are stable but library callers must update imports and, ideally, migrate to the Provider API.

## When to Use / When Not

**Use when:**
- Your stack is Go and you want migrations that live in the same repo and binary as the app.
- You need migration logic that runs Go code (backfills, data transforms) alongside SQL.
- You want a small, transparent tool whose entire behavior you can hold in your head.
- You deploy a single binary and want migrations embedded via `embed.FS`.

**Avoid when:**
- You want declarative schema-as-code with automatic diffing and drift detection — use Atlas.
- You need enforced checksums / tamper detection on applied migrations — use Flyway/Liquibase.
- Your team is not Go and Go-function migrations bring no value — a language-agnostic tool is simpler.
- You rely heavily on automated, verified rollbacks — goose's down story is manual and unguarded.

## Alternatives

- golang-migrate/migrate — the closest competitor; CLI + library, very broad driver and source list, SQL-only migrations. Use it when you want more drivers/sources and don't need Go-function migrations.
- amacneil/dbmate — language-agnostic, single binary, SQL-only, framework-neutral. Use it when your app isn't Go or you want a tool shared across polyglot services.
- ariga/atlas — declarative schema-as-code with a diff engine, HCL, and migration linting. Use it when you want the tool to compute migrations from a desired-state schema.
- rubenv/sql-migrate — Go library with SQL migrations and packr/embed support. Use it when you want something goose-like tied into Go config conventions.
- jackc/tern — Postgres-only migration tool from the pgx author. Use it when you are all-in on Postgres and want tight pgx integration.

## History

| Version | Date | Notes |
|---------|------|-------|
| goose (original) | 2012 | Created by Liam Staskawicz[^1]. |
| pressly/goose | 2016 | Adopted and maintained under the pressly org; repo created[^1]. |
| v3 | ~2019 | New module path `.../v3`, pluggable dialects, context-aware API. |
| v3 (later) | 2022–2024 | `NewProvider` non-global API[^3], `.env` loading, `ENVSUB` interpolation, expanded driver set (Turso, StarRocks, YDB, Spanner). |

## References

[^1]: pressly/goose repository and LICENSE header ("Original work Copyright (c) 2012 Liam Staskawicz"). https://github.com/pressly/goose
[^2]: Repository metadata (stars, forks, last push) via GitHub API, 2026-07: ~11.2k stars, ~672 forks, MIT-licensed, last push 2026-07-11. https://github.com/pressly/goose
[^3]: goose Provider API reference. https://pkg.go.dev/github.com/pressly/goose/v3#NewProvider
[^4]: goose hybrid versioning discussion. https://github.com/pressly/goose/issues/63#issuecomment-428681694

## Tags

go, golang, database-migrations, schema-migration, sql, cli, library, postgres, mysql, sqlite, database-tooling
