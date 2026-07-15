# sqldelight/sqldelight

> Generates typesafe Kotlin APIs from hand-written SQL, verifying schema, queries, and migrations at compile time.

[GitHub repo](https://github.com/sqldelight/sqldelight) ·
[Official website](https://sqldelight.github.io/sqldelight/) ·
[License: Apache-2.0](https://github.com/sqldelight/sqldelight/blob/master/LICENSE.txt)

## Overview

SQLDelight is an anti-ORM. Instead of mapping objects to tables behind annotations
or a DSL, you write `.sq` files containing ordinary SQL — `CREATE TABLE`, labeled
`SELECT`/`INSERT`/`UPDATE` statements, and `.sqm` migrations — and a Gradle plugin
generates typed Kotlin functions from them at build time. The SQL is the source of
truth; the Kotlin is derived. The parser understands your schema, so a typo in a
column name or a query that references a dropped table is a compile error, not a
runtime crash[^1].

It began at Square in 2016 (originally under `com.squareup.sqldelight`), moved to
Cash App's `app.cash.sqldelight` namespace with the 2.0 rewrite, and now lives in
its own `sqldelight` GitHub organization[^2]. Its defining bet is Kotlin
Multiplatform: the same `.sq` definitions generate a shared data layer that runs on
Android, iOS/native, JVM, and JavaScript through pluggable `SqlDriver`
implementations. That makes it the default persistence choice for KMP apps that want
to share a database module across platforms.

The tension is philosophical and practical. You get exact, auditable SQL and no
reflection, but you also get no relationship graph, no lazy loading, and no query
builder — dynamic queries are awkward, and you own every `JOIN`. Teams that want an
ORM's conveniences will find SQLDelight deliberately spartan; teams that distrust
ORMs find that the point.

## Getting Started

Apply the Gradle plugin and declare a database:

```kotlin
// build.gradle.kts
plugins {
  id("app.cash.sqldelight") version "2.0.2"
}

sqldelight {
  databases {
    create("Database") {
      packageName.set("com.example")
    }
  }
}
```

Write SQL in `src/main/sqldelight/com/example/Player.sq`:

```sql
CREATE TABLE hockeyPlayer (
  id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL
);

selectAll:
SELECT * FROM hockeyPlayer;

insertPlayer:
INSERT INTO hockeyPlayer(name) VALUES (?);
```

The build generates a `Database` and a `PlayerQueries` class:

```kotlin
val driver = AndroidSqliteDriver(Database.Schema, context, "players.db")
val database = Database(driver)

database.playerQueries.insertPlayer("Wayne Gretzky")
val players: List<HockeyPlayer> = database.playerQueries.selectAll().executeAsList()
```

## Architecture / How It Works

SQLDelight is built on JetBrains' IntelliJ platform PSI and grammar-kit tooling —
the same machinery that powers the IDE plugin. The Gradle plugin parses each `.sq`
file into a syntax tree against a dialect-specific grammar, resolves every statement
against the declared schema, and emits Kotlin via a code generator. Because the
parser is a real SQL grammar and not string matching, it can offer autocomplete,
find-usages, and refactoring inside Android Studio / IntelliJ through the companion
plugin[^1].

Generated code centers on a few shapes:

- Each labeled statement becomes a function. `SELECT`s return a `Query<T>` you
  drain with `.executeAsList()`, `.executeAsOne()`, `.executeAsOneOrNull()`, or
  observe as a Kotlin `Flow` via the `coroutines-extensions` artifact.
- Column adapters (`ColumnAdapter<T, S>`) map custom Kotlin types to SQL storage
  types; you wire them when constructing the database.
- Migrations live in `.sqm` files and are verified at compile time against a
  captured schema, catching statements that don't apply cleanly.

At runtime the only abstraction is `SqlDriver` (with `SqlPreparedStatement` and
`SqlCursor`). Platform drivers are separate dependencies: `AndroidSqliteDriver`
(wrapping AndroidX SQLite), `NativeSqliteDriver` (backed by SQLiter for
iOS/macOS/Windows), `JdbcSqliteDriver` / `JdbcDriver` on the JVM, and a web-worker
driver for JavaScript. The 2.0 line added async driver support so the JS/web target
can await results rather than block[^2].

Dialects are also modular. SQLite is the mature default; MySQL, PostgreSQL, and
HSQL/H2 are separate dialect artifacts and are JVM-only. Compile-time checking
validates SQL against the grammar and declared schema — it does not run against a
live database, so dialect-specific runtime behavior can still differ from what the
grammar accepts.

## Production Notes

**It is not an ORM, by design.** No lazy loading, no relationship traversal, no
identity map. You write joins and denormalize result rows yourself. Dynamic queries
(variable `WHERE` clauses, `IN` lists of unknown length) are the weakest ergonomic
area; teams often generate SQL for the common shapes and drop to raw driver calls
for the rest.

**Change notifications are table-level.** A `Query` observed as a `Flow` is
re-run when any statement touches a listed table — invalidation is coarse, not
row-scoped, so a hot table can re-emit more often than a fine-grained observer would.

**Threading is yours to own.** The drivers run queries on the calling thread.
SQLDelight ships no connection pool; on the JVM you pair `JdbcSqliteDriver` (or a
Postgres/MySQL `JdbcDriver`) with HikariCP or similar, and you dispatch work off the
main thread with coroutines yourself.

**Migrations need captured schema.** Compile-time migration verification depends on
storing schema snapshots (the `generateSqlDelightSchema` / verify tasks). Skipping
that step means migrations aren't actually checked against the prior schema.

**The 1.x → 2.0 upgrade is a real migration, not a bump.** The package moved from
`com.squareup.sqldelight` to `app.cash.sqldelight`, the Gradle plugin id changed,
dialects and drivers were split into separate dependencies, and several runtime APIs
changed[^2]. Expect import rewrites across the whole data layer and a coordinated
Gradle config change.

**IDE plugin lag.** The IntelliJ/Android Studio plugin occasionally trails new IDE
releases, so editor features can break for a window after an IDE update even when the
Gradle build is fine.

**Native memory model history.** Early Kotlin/Native forced strict memory-model
constraints on the iOS driver; the modern Kotlin/Native memory manager removed most
of that friction, but codebases pinned to old Kotlin versions may still hit it.

## When to Use / When Not

**Use when:**
- You're building Kotlin Multiplatform and want one SQL-defined data layer shared
  across Android, iOS, JVM, and web.
- You prefer owning explicit SQL over annotation- or DSL-based mapping, and want that
  SQL checked at compile time.
- You want a small runtime with no reflection and predictable generated code.

**Avoid when:**
- You want ORM conveniences — relationship graphs, lazy loading, cascade mapping.
- Your queries are heavily dynamic and assembled at runtime.
- You're server-side on PostgreSQL/MySQL and want a mature, feature-complete query
  layer (jOOQ/Exposed/Hibernate cover more of those dialects).
- You're not in the Kotlin ecosystem.

## Alternatives

- androidx/room — Google's annotation-based Android ORM, now with KMP support; use when you want first-party Android tooling and are fine with less direct SQL ownership.
- JetBrains/Exposed — Kotlin SQL DSL + lightweight DAO for the JVM; use for server-side PostgreSQL/MySQL where a query builder beats hand-written `.sq` files.
- jOOQ/jOOQ — typesafe SQL generated from an existing database schema; use on the JVM when the database is the source of truth and you target rich SQL dialects.
- ktorm/ktorm — Kotlin ORM/DSL for the JVM; use when you want an entity-and-DSL model rather than raw SQL, server-side only.
- realm/realm-kotlin — object database for mobile; use when you want an object store and reactive queries instead of SQL and drivers.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016 | Released at Square under `com.squareup.sqldelight`[^2]. |
| 1.0 | 2019 | First stable line; Kotlin code generation, multiplatform drivers. |
| 2.0.0 | 2023-08 | Rewrite: `app.cash.sqldelight` namespace, modular dialects, async drivers, new Gradle plugin id[^2]. |
| 2.0.x | 2024 | Patch line under the `sqldelight` GitHub organization. |

## References

[^1]: SQLDelight project website — overview, dialects, and IDE features. https://sqldelight.github.io/sqldelight/
[^2]: SQLDelight 2.0 upgrade / changelog notes on the namespace move and modular dialects. https://sqldelight.github.io/sqldelight/2.0.2/upgrading/

## Tags

kotlin, kotlin-multiplatform, sql, sqlite, code-generation, orm-alternative, android, database, compile-time, gradle-plugin, typesafe
