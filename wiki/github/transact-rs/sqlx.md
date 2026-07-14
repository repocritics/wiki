# transact-rs/sqlx

> An async, pure-Rust SQL toolkit with compile-time-checked queries that verifies your SQL against a real database instead of a Rust query DSL.

[GitHub repo](https://github.com/launchbadge/sqlx) ·
[docs.rs](https://docs.rs/sqlx) ·
[License: Apache-2.0 OR MIT](https://github.com/launchbadge/sqlx/blob/main/LICENSE-APACHE)

## Overview

SQLx is a database access crate for Rust built around async/await from the ground up, originally authored by the LaunchBadge consultancy and first published to crates.io in 2020[^1]. The repository now lives under the `transact-rs` organization (redirected from `launchbadge/sqlx`). It supports PostgreSQL, MySQL/MariaDB, and SQLite, and works across the `tokio`, `async-std`, and `actix` runtimes with either `native-tls` or `rustls` for transport security.

Its defining idea is captured by the project's own slogan, "SQLx is not an ORM." It offers no query builder or DSL; you write raw SQL strings. The `query!` / `query_as!` macros then achieve compile-time verification by connecting to a live development database at build time and asking the database itself to describe the query — types, arity, and (partial) nullability. This trades the schema-as-Rust safety of an ORM like Diesel for the freedom to use any SQL your database accepts, including extension syntax the crate never has to parse[^2].

That tradeoff is also the source of most friction. The compile-time checking that is SQLx's headline feature couples your build to a running database (or a committed offline cache), and its nullability inference is necessarily imperfect. The crate is "pure Rust" for Postgres and MySQL — `#![forbid(unsafe_code)]` — but the SQLite driver links the C `libsqlite3` library through `libsqlite3-sys` and is exempt from that guarantee[^3].

## Getting Started

```toml
# Cargo.toml — pick one runtime + optional TLS + one or more drivers
[dependencies]
sqlx = { version = "0.8", features = [
    "runtime-tokio", "tls-rustls-ring-webpki",
    "postgres", "macros", "migrate", "uuid"
] }
```

```rust
use sqlx::postgres::PgPoolOptions;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect("postgres://postgres:password@localhost/test")
        .await?;

    // Compile-time checked: requires DATABASE_URL (or an offline .sqlx cache) at build time.
    let rec = sqlx::query!("SELECT id, email FROM users WHERE id = $1", 1_i64)
        .fetch_one(&pool)
        .await?;

    println!("{} {}", rec.id, rec.email);
    Ok(())
}
```

Use `sqlx::query()` (a `&str`, no macro) when you want runtime queries with no build-time database, and `query!`/`query_as!` when you want the compiler to validate SQL against your schema.

## Architecture / How It Works

SQLx is organized into a driver-per-database model behind shared `Executor`, `Connection`, `Row`, `Encode`/`Decode`, and `Type` traits. Each backend (`sqlx-postgres`, `sqlx-mysql`, `sqlx-sqlite`) implements the wire protocol directly; the Postgres and MySQL drivers speak the binary protocol in pure Rust, while SQLite calls into the bundled C library. `sqlx::Pool` provides connection pooling, and the high-level `query` API automatically prepares and caches statements per connection.

The compile-time macros are a separate world from the runtime API. `query!` expands in a proc-macro (`sqlx-macros`) that opens a connection to `DATABASE_URL` during compilation, runs the database's describe/prepare path, and generates an anonymous record struct whose fields mirror the result columns. Because the database does the analysis, SQLx never parses your SQL — but it also inherits each backend's limits on what it will report. Nullability, in particular, is under-determined: the database frequently cannot say whether a projected column is nullable (joins, expressions, aggregates), so SQLx conservatively wraps values in `Option` or requires you to override with the `column as "name!"` (force non-null) and `as "name?"` (force nullable) annotations[^2].

To avoid needing a database at every build, SQLx supports **offline mode**: `cargo sqlx prepare` (from `sqlx-cli`) serializes query metadata into a `.sqlx/` directory (historically a single `sqlx-data.json`). Committed to the repo, this cache lets CI and Docker builds compile without database connectivity — as long as it stays in sync with the actual schema and query set.

The `Any` driver (`AnyPool`) abstracts over backends chosen at runtime by URL scheme, at the cost of losing driver-specific types. MSSQL was supported prior to 0.7 but was removed pending a full driver rewrite under the paid "SQLx Pro" initiative[^4].

## Production Notes

**RUSTSEC-2024-0363: upgrade to ≥ 0.8.1.** Versions before 0.8.1 mis-sized a protocol length prefix, so a sufficiently large bind parameter could overflow the length field and cause the database to reinterpret attacker-controlled bytes as a new query — a query-truncation / injection vector even with parameterized queries[^5]. This is the single most important operational fact about SQLx pinning.

**The build database is a real dependency.** Any change that touches a `query!` invocation needs either `DATABASE_URL` pointing at a schema-matching database at compile time, or a current `.sqlx` offline cache. Forgetting to run `cargo sqlx prepare` after a query or migration change produces builds that pass locally (live DB) but fail in CI (stale cache), or vice versa. Teams typically add a CI check that `cargo sqlx prepare --check` reports no drift.

**Nullability annotations are unavoidable at scale.** Real queries with `LEFT JOIN`s, `COALESCE`, and computed columns routinely produce wrong `Option`-ness until you sprinkle `as "col!"` / `as "col?"` overrides. This is inherent to describing SQL via the database rather than a schema model.

**PgBouncer in transaction-pooling mode conflicts with statement caching.** SQLx prepares and caches statements per connection; a transaction-mode pooler that hands out different server connections per statement breaks that assumption. Set the pool's `statement_cache_capacity(0)` (or use a session-mode pooler / direct connection) when fronting Postgres with PgBouncer.

**SQLite is not pure Rust and links C.** The default `sqlite` feature statically bundles SQLite; `sqlite-unbundled` links the system library (needs SQLite ≥ 3.20 and `bindgen`, which adds build time and can produce link errors on old systems). If your threat model requires `forbid(unsafe_code)` end-to-end, the SQLite backend does not qualify.

**Build times.** Compile-time query analysis is not free. The README recommends `[profile.dev.package.sqlx-macros] opt-level = 3` to speed up incremental `cargo check`/`build`.

**Upgrades are frequently breaking.** 0.6 → 0.7 dropped `async-std` as the default runtime (you must now select a `runtime-*` feature explicitly) and removed MSSQL; 0.7 → 0.8 changed several type mappings and driver behaviors. Minor-version bumps carry non-trivial migration notes in the changelog — read them.

## When to Use / When Not

**Use when:**
- You want to write real SQL and still catch schema mistakes at compile time.
- You need async access to Postgres, MySQL/MariaDB, or SQLite from Rust.
- You want connection pooling, embedded migrations (`migrate!`), and runtime/TLS flexibility without an ORM's abstraction.

**Avoid when:**
- You want a typed query builder or schema-as-Rust safety — reach for an ORM.
- You cannot provide a build-time database or maintain an offline cache in CI.
- You need a database SQLx does not support (Oracle, MSSQL today, etc.).
- You require `forbid(unsafe_code)` and your target is SQLite.

## Alternatives

- diesel-rs/diesel — synchronous ORM and typed query DSL (async via `diesel-async`); use when you want schema-as-Rust and a compile-time query builder rather than raw SQL.
- SeaQL/sea-orm — async ORM built on top of SQLx; use when you want entities, relations, and an active-record style rather than hand-written SQL.
- sfackler/rust-postgres — lower-level sync/async Postgres-only driver (`tokio-postgres`); use when you only target Postgres and want no macros or build-time database.
- SeaQL/sea-query — standalone SQL query builder; use when you want programmatic query construction without a full ORM.
- rbatis/rbatis — async ORM/SQL toolkit; use when you prefer a MyBatis-style mapper approach.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2020 | Initial crates.io release by LaunchBadge[^1]. |
| 0.5.x | 2020–2021 | Broad driver maturation; offline mode cache. |
| 0.6.0 | 2022-06 | API refinements; MSSQL still present. |
| 0.7.0 | 2023-06 | MSSQL removed; `async-std` no longer the default runtime[^4]. |
| 0.8.0 | 2024-07 | Type-mapping and driver changes; new TLS feature combinations. |
| 0.8.1 | 2024-08 | Security fix for RUSTSEC-2024-0363 (length-prefix overflow)[^5]. |

## References

[^1]: SQLx crate on crates.io. https://crates.io/crates/sqlx
[^2]: "SQLx is not an ORM!" and compile-time verification, project README. https://github.com/launchbadge/sqlx#sqlx-is-not-an-orm
[^3]: Safety section (`forbid`/`deny` unsafe, SQLite exemption), project README. https://github.com/launchbadge/sqlx#safety
[^4]: SQLx Pro initiative (MSSQL driver rewrite) discussion. https://github.com/launchbadge/sqlx/discussions/1616
[^5]: RUSTSEC-2024-0363 — Binary protocol misinterpretation caused by truncating or overflowing casts, fixed in SQLx 0.8.1. https://rustsec.org/advisories/RUSTSEC-2024-0363.html

## Tags

rust, sql, database, async, postgresql, mysql, sqlite, compile-time-checked, connection-pool, database-driver, tokio, no-orm
