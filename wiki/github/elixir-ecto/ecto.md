# elixir-ecto/ecto

> Elixir's data-mapping and query toolkit — a functional take on persistence that is deliberately not an ORM.

[GitHub repo](https://github.com/elixir-ecto/ecto) ·
[Official docs](https://hexdocs.pm/ecto) ·
[License: Apache-2.0](https://github.com/elixir-ecto/ecto/blob/master/LICENSE)

## Overview

Ecto is the standard database layer for Elixir, first released by Plataformatec in 2013 and maintained by Dashbit and the elixir-ecto team since around 2020[^1]. It is composed of four largely independent pieces: schemas (struct definitions with types), changesets (validation and cast pipelines), a composable query DSL built on Elixir macros, and repositories (the `Repo` boundary that actually talks to a database). At roughly 6.5k stars it is not a large-numbers project by JavaScript standards, but within Elixir it is near-universal — Phoenix scaffolds against it by default and most production Elixir apps depend on it.

The defining design choice is that Ecto is explicitly **not** an ORM. There is no identity map, no lazy loading, no dirty tracking, and no implicit persistence. A schema is a plain struct; a changeset is a data structure describing a proposed change; nothing hits the database until you hand a query or changeset to `Repo`. Associations are never auto-loaded — accessing an unloaded association returns a `%Ecto.Association.NotLoaded{}` marker rather than issuing a query. This eliminates the N+1-by-accident class of bugs that plague ActiveRecord-style ORMs, at the cost of forcing the developer to be explicit about every preload and every write.

The second defining trait is the split between `ecto` and `ecto_sql`. The core `ecto` package contains schemas, changesets, the query DSL, and the `Repo` behaviour but no SQL adapter. Actually talking to Postgres/MySQL/etc. requires the separate `ecto_sql` package plus a driver. This lets Ecto model non-database data sources (embedded schemas, API payloads, form params) without pulling in a SQL stack, and it is a frequent source of first-day confusion for newcomers who add `:ecto` and find they cannot connect to Postgres.

## Getting Started

```elixir
# mix.exs — you almost always want ecto_sql + a driver, not bare ecto
defp deps do
  [
    {:ecto_sql, "~> 3.12"},
    {:postgrex, ">= 0.0.0"}
  ]
end
```

```elixir
# lib/my_app/repo.ex
defmodule MyApp.Repo do
  use Ecto.Repo, otp_app: :my_app, adapter: Ecto.Adapters.Postgres
end

# lib/my_app/weather.ex
defmodule MyApp.Weather do
  use Ecto.Schema
  import Ecto.Changeset

  schema "weather" do
    field :city, :string
    field :temp_lo, :integer
    field :prcp, :float, default: 0.0
    timestamps()
  end

  def changeset(weather, attrs) do
    weather
    |> cast(attrs, [:city, :temp_lo, :prcp])
    |> validate_required([:city])
  end
end
```

```elixir
# Querying — keyword and pipe syntax are equivalent
import Ecto.Query

MyApp.Weather
|> where([w], w.prcp > 0 or is_nil(w.prcp))
|> order_by(:temp_lo)
|> limit(10)
|> MyApp.Repo.all()
```

Run `mix ecto.gen.migration`, `mix ecto.migrate`, and `mix ecto.create` to manage schema (migration tooling lives in `ecto_sql`).

## Architecture / How It Works

The query DSL is macro-based and compiles at build time. `from`, `where`, `select`, and friends are macros that expand into an `%Ecto.Query{}` struct rather than string concatenation; field references like `w.prcp` are checked against known schema fields at compile time when a schema is in scope. Because queries are data, they compose — you can build a base query in one function and refine it in another, and fragments interpolated with `^value` are always sent as parameterized bindings, so the DSL is SQL-injection-safe by construction. Raw SQL escape hatches (`fragment/1`, `Repo.query/2`) exist for cases the DSL cannot express.

Changesets are the write-path counterpart. `cast/3` filters external params against an allowlist of fields (mass-assignment protection), coerces types, and accumulates errors; `validate_*` and `*_constraint` functions layer on rules. Crucially, `unique_constraint`, `foreign_key_constraint`, and `check_constraint` do not query the database to validate — they annotate the changeset so that when the write fails at the database level, the `Postgrex`/`MyXQL` error is translated into a friendly changeset error instead of an exception. This pushes the source of truth for integrity down to the database where it belongs and avoids TOCTOU race conditions.

`Repo` is the only component that performs I/O. It is a behaviour; `use Ecto.Repo` generates functions (`all`, `get`, `insert`, `update`, `transaction`, `preload`…) that delegate to a configured adapter. Connection pooling is handled by DBConnection (via the driver), not by Ecto itself. `Ecto.Multi` is the composable transaction primitive: it builds an ordered list of named operations as a data structure that runs inside a single transaction and short-circuits on the first failure, returning which step failed — the idiomatic way to express multi-step writes without nesting.

The `ecto` / `ecto_sql` split runs deep: adapters implement behaviours (`Ecto.Adapter`, `Ecto.Adapter.Queryable`, `Ecto.Adapter.Schema`) so third-party backends (SQLite via `ecto_sqlite3`, ClickHouse via `ecto_ch`, ETS via `etso`) can plug in without touching core. Not every adapter supports every feature — migrations, transactions, and certain query constructs vary by backend.

## Production Notes

- **Preloading is manual and it matters.** Forgetting `Repo.preload/2` (or a `preload:` in the query) surfaces as `NotLoaded` structs, not silent extra queries — which is safer but means preload strategy is your job. Prefer join-based preloads (`preload: [posts: p]`) over separate-query preloads when filtering the association, or you will fetch more rows than you filtered on.
- **`Repo.all` has no implicit limit.** A query with a bad or missing `where` will happily stream your whole table into memory. Use `Repo.stream/2` inside a transaction for large result sets; the default `all` materializes everything.
- **Migrations run in a transaction by default, except when they can't.** Postgres `CREATE INDEX CONCURRENTLY` cannot run inside a transaction — you must set `@disable_ddl_transaction true` and `@disable_migration_lock true` in that migration, a well-known footgun for zero-downtime index creation on large tables.
- **`prepare: :unnamed` vs prepared-statement caching.** Behind PgBouncer in transaction-pooling mode, Postgres prepared statements break; you need `prepare: :unnamed` in the Repo config (or a pooler in session mode). This bites teams moving to managed Postgres with connection poolers.
- **Sandbox concurrency in tests.** `Ecto.Adapters.SQL.Sandbox` gives each test its own transaction rolled back at teardown, enabling async tests — but any code path that spawns a process (Tasks, GenServers) must be granted access via `allow/3` or it will not see the sandboxed connection. This is the single most common Ecto test-setup question.
- **Version support is narrow.** Only the latest minor gets bug fixes; older 3.x lines get security patches only (see the support table in the README). The 3.x API has been stable since 2018, so upgrades are usually low-drama, but do not expect backports.

## When to Use / When Not

**Use when:**
- You are building on Elixir/Phoenix — it is the default and the ecosystem assumes it.
- You want explicit, composable queries and validation without ORM magic or hidden queries.
- You need the same validation/casting machinery for both DB rows and non-DB data (form params, embedded documents, API payloads).
- Correctness-critical writes benefit from `Ecto.Multi` and database-enforced constraints.

**Avoid when:**
- You are not on the BEAM — Ecto is Elixir-only and does not port.
- You want an ActiveRecord-style ORM with lazy loading and automatic persistence; Ecto's explicitness will feel like friction.
- Your data store has no adapter and you are unwilling to write one against the adapter behaviours.
- You need a document/graph store as a first-class model — Ecto is relational-shaped even where adapters exist.

## Alternatives

- rails/rails (ActiveRecord) — use instead when you are in Ruby and want convention-over-configuration ORM with lazy loading.
- prisma/prisma — use instead in the TypeScript/Node world when you want a schema-first ORM with generated types.
- go-gorm/gorm — use instead in Go for a struct-tag ORM with association auto-loading.
- launchbadge/sqlx — use instead in Rust when you want compile-time-checked raw SQL rather than a query DSL.
- dashbit/broadway or plain Postgrex — use instead when you need streaming ingestion or want to drop below the schema/changeset layer entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2014-09 | First public release from Plataformatec. |
| 1.0 | 2015-09 | Stable changeset/query API. |
| 2.0 | 2016-06 | `Ecto.Multi`, concurrent SQL sandbox, subqueries. |
| 3.0 | 2018-10 | Core split into `ecto` + `ecto_sql`; DBConnection 2.0; API declared stable[^2]. |
| 3.x | 2019–2026 | Incremental: windows/CTEs, `Repo.stream`, dynamic queries, JSON improvements. Actively maintained; latest push 2026-07-09. |

## References

[^1]: Ecto README and repository, elixir-ecto/ecto (created 2013-06-12; Apache-2.0; ~6.5k stars, ~1.5k forks as of 2026-07). https://github.com/elixir-ecto/ecto
[^2]: "Ecto 3.0 released", Dashbit / José Valim — core split into `ecto` and `ecto_sql`. https://hexdocs.pm/ecto/Ecto.html and https://dashbit.co/blog
[^3]: Official documentation and getting-started guide. https://hexdocs.pm/ecto

## Tags

elixir, database, orm, query-builder, data-mapper, sql, postgresql, changesets, functional, beam, persistence
