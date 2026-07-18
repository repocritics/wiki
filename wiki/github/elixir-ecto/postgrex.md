# elixir-ecto/postgrex

> The PostgreSQL driver for Elixir — the binary-protocol workhorse underneath
> every Ecto/Postgres application on the BEAM.

[GitHub repo](https://github.com/elixir-ecto/postgrex) ·
[Documentation](http://hexdocs.pm/postgrex/) ·
License: Apache-2.0[^1]

## Overview

Postgrex is the de facto PostgreSQL client for Elixir, started by Eric
Meadows-Jönsson (Elixir core team, co-creator of Hex) in 2013[^1]. It is the
driver behind `ecto_sql`'s Postgres adapter — nearly every Elixir application
that talks to Postgres runs it, knowingly or not — and it works standalone
for raw SQL without Ecto.

The defining design decision: Postgrex speaks PostgreSQL's **binary wire
protocol**, not the text protocol most client libraries use[^2]. Binary
encoding is more efficient (a 4-byte integer travels as 4 bytes, not a
digit string) and gives arrays and composite types for free, but it surfaces
details text-protocol drivers paper over: OID types must be passed as
integers or explicitly cast, and there is no implicit casting of any kind —
a string where a date is expected is an error, not a coercion.

Maintenance is healthy: ~1.2k stars, pushes within days (last 2026-07-15),
and only 4 open issues — low because the surface is stable and feature
discussion happens at the Ecto layer. Notably, Postgrex is still on 0.x after
13 years (latest 0.22.3, 2026-07-09[^3]); a 1.0.0-rc was cut in 2016 and
abandoned. In practice 0.x is treated as stable, with minor versions as the
compatibility boundary.

## Getting Started

```elixir
# mix.exs deps
{:postgrex, "~> 0.22"}
```

```elixir
{:ok, pid} =
  Postgrex.start_link(
    hostname: "localhost",
    username: "postgres",
    password: "postgres",
    database: "postgres"
  )

Postgrex.query!(pid, "SELECT user_id, text FROM comments WHERE user_id = $1", [3])
#=> %Postgrex.Result{command: :select, rows: [[3, "hey"]], num_rows: 1, ...}
```

Parameters are always `$1`-style placeholders; every query is prepared under
the hood. With Ecto you never call this API — `Ecto.Adapters.Postgres` does.

## Architecture / How It Works

Postgrex is a thin protocol implementation layered on
**elixir-ecto/db_connection**[^4], which owns pooling, checkout, queueing,
and backoff for the whole Ecto driver family (`myxql`, `tds`, `exqlite`).
Understanding Postgrex operationally is mostly understanding DBConnection.

- **Type system.** Encoding/decoding is done by *extensions* — one module per
  Postgres type (`lib/postgrex/extensions/`)[^2]. `Postgrex.Types.define/3`
  compiles extensions into a type module at build time; at connect time the
  driver bootstraps OID mappings from `pg_type`. Custom types (PostGIS etc.)
  are extensions plus a custom types module passed via `types:`.
- **Prepared statements.** Queries are named-prepared and cached per
  connection. `prepare: :unnamed` forces unnamed statements for pooler
  compatibility (see Production Notes).
- **`Postgrex.stream/4`** — cursor-based streaming of large result sets,
  usable only inside a transaction.
- **`Postgrex.Notifications`** — LISTEN/NOTIFY on a dedicated connection
  process, separate from the query pool.
- **`Postgrex.SimpleConnection` / `Postgrex.ReplicationConnection`** (since
  0.16, 2022)[^5] — single-connection behaviours outside the pool. The
  latter speaks the logical replication protocol and is the foundation the
  Elixir CDC ecosystem builds on rather than reimplementing it.

Mapping is struct-faithful rather than convenient: `numeric` decodes to
`Decimal` (a required dependency), `interval` to `Postgrex.Interval` (or
Elixir 1.17's `Duration` when opted in), and anonymous composite types
decode to tuples but cannot be encoded back[^2].

## Production Notes

- **The OID footgun.** Because of the binary protocol, `regclass` and other
  OID types cannot take a string parameter: `query("select nextval($1)",
  ["some_sequence"])` fails. Cast explicitly (`$1::text::regclass`) or
  resolve the OID once and reuse it[^2]. This is the single most common
  "works in psql, fails in Postgrex" report.
- **PgBouncer.** Older than 1.21.0 in transaction/statement pooling mode,
  named prepared statements break (requests route to different backends).
  Set `prepare: :unnamed`, which re-parses every query; 1.21+ supports named
  statements natively[^6].
- **Pool tuning is DBConnection tuning.** Since 0.14 (2018), `:pool_timeout`
  is gone; overload behaviour is governed by `:queue_target` and
  `:queue_interval`[^7] — sustained checkout pressure makes DBConnection
  shed load with `DBConnection.ConnectionError`. These errors are routinely
  misread as database failures; they are client-side queue pressure.
- **JSON library is a compile-time choice.** Jason is the default; switching
  via `config :postgrex, :json_library` requires `mix deps.clean postgrex
  --build` — a classic source of "config change had no effect" confusion[^2].
- **Timezone flattening.** All timestamps come back UTC or assumed-UTC;
  session-timezone semantics must be handled in application code.
- **ReplicationConnection is single-process.** No pooling, delivery
  semantics are on you (acknowledge LSNs yourself), and pre-0.20 versions
  had reconnect edge cases[^3]. A building block, not a turnkey CDC system.

## When to Use / When Not

**Use when:**
- You are on the BEAM and talking to PostgreSQL — this is the default, and
  the default is correct.
- You need raw SQL without Ecto's mapping layer (ETL scripts, simple
  services).
- You need LISTEN/NOTIFY or logical replication from Elixir;
  `ReplicationConnection` is the only maintained option.

**Avoid when:**
- You want schemas, migrations, changesets, and a query DSL — use Ecto
  (`ecto_sql`); dropping to the driver is rarely a win for application code.
- You are on Erlang without Elixir tooling — Erlang-native drivers avoid
  pulling in the Elixir compile chain.
- You expect implicit type coercion or text-protocol semantics; Postgrex
  deliberately refuses both.

## Alternatives

- elixir-ecto/ecto_sql — use instead when you want the full data-mapping
  toolkit (schemas, migrations, query DSL); it uses Postgrex internally.
- epgsql/epgsql — use instead from pure Erlang projects; the longest-lived
  Erlang Postgres driver.
- erleans/pgo — use instead for a minimal Erlang client with a built-in pool
  and no Elixir dependency.
- semiocast/pgsql — older Erlang alternative; mostly of historical interest.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2013-10 | Initial release by Eric Meadows-Jönsson[^3]. |
| 1.0.0-rc.1 | 2016-11 | 1.0 attempt during the Ecto 2.0 era; abandoned, versioning returned to 0.x[^3]. |
| 0.14.0 | 2018-10 | DBConnection 2.0; `:pool_timeout` replaced by `:queue_target`/`:queue_interval`[^7]. |
| 0.16.0 | 2022-01 | `Postgrex.SimpleConnection` and `Postgrex.ReplicationConnection`[^5]. |
| 0.17.0 | 2023-04 | SCRAM server signature verification; multirange support[^3]. |
| 0.18.0 | 2024-05 | Elixir 1.17 `Duration` encoding/decoding for intervals[^3]. |
| 0.22.3 | 2026-07 | Latest release; SCRAM-SHA-256-PLUS in development for 0.23[^3]. |

## References

[^1]: Postgrex README, license section (Apache-2.0, copyright 2013 Eric Meadows-Jönsson). https://github.com/elixir-ecto/postgrex#license
[^2]: Postgrex README — data representation, OID type encoding, extensions, JSON support. https://github.com/elixir-ecto/postgrex
[^3]: Postgrex CHANGELOG and hex.pm release history. https://github.com/elixir-ecto/postgrex/blob/master/CHANGELOG.md · https://hex.pm/packages/postgrex
[^4]: DBConnection — database connection behaviour and pooling for Elixir drivers. https://github.com/elixir-ecto/db_connection
[^5]: Postgrex v0.16.0 changelog entry (2022-01-23). https://github.com/elixir-ecto/postgrex/blob/master/CHANGELOG.md
[^6]: PgBouncer 1.21.0 release notes — prepared statement support. https://www.pgbouncer.org/changelog.html#pgbouncer-121x
[^7]: Postgrex v0.14.0 changelog entry (2018-10-29); DBConnection queue options. https://hexdocs.pm/db_connection/DBConnection.html

## Tags

elixir, postgresql, database-driver, binary-protocol, ecto, db-connection, connection-pooling, logical-replication, listen-notify, beam
