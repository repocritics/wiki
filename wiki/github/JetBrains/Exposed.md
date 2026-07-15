# JetBrains/Exposed

> A Kotlin SQL library offering two layers over JDBC/R2DBC: a typesafe SQL DSL and a lightweight Active-Record-style DAO.

[GitHub repo](https://github.com/JetBrains/Exposed) ·
[Official website](https://www.jetbrains.com/exposed/) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

Exposed is JetBrains' in-house SQL library for Kotlin, on GitHub since 2013[^1] —
predating Kotlin 1.0 and making it one of the oldest Kotlin data-access projects
still maintained. It is deliberately not a full JPA-style ORM. Instead it gives
you two independent access styles over the same core: a **DSL** that mirrors SQL
as typesafe Kotlin (`Users.selectAll().where { … }`) and a **DAO** layer that
wraps rows in entity objects (`User.new { … }`, `user.city`). You can use either
in isolation, or the DSL alone with no entity machinery at all.

The defining tension is that Exposed sits between a query builder and an ORM
without fully committing to either. The DSL is thin, explicit, and predictable —
what you write is close to the SQL emitted. The DAO adds convenience
(lazy references, `referrersOn`) but reintroduces classic ORM footguns: entities
are bound to their `transaction` scope and lazy relationships load per-access,
so N+1 patterns and "entity accessed outside transaction" errors are the usual
first surprises. Teams that treat Exposed as a query builder (DSL-first, map to
plain data classes) tend to hit fewer of these.

The project reached its **1.0.0** milestone after more than a decade on `0.x`
versions. That release repackaged everything under `org.jetbrains.exposed.v1.*`
and added R2DBC (reactive/non-blocking) support alongside the long-standing JDBC
path[^2]. It is an official JetBrains project (Apache-2.0), tracked in JetBrains'
YouTrack rather than GitHub Issues — the GitHub open-issue count understates the
real backlog, which lives at youtrack.jetbrains.com/issues/EXPOSED[^3].

## Getting Started

```kotlin
// build.gradle.kts — check Maven Central for the current version
dependencies {
    implementation("org.jetbrains.exposed:exposed-core:1.0.0")
    implementation("org.jetbrains.exposed:exposed-jdbc:1.0.0")
    implementation("com.h2database:h2:2.2.224")
}
```

```kotlin
import org.jetbrains.exposed.v1.core.Table
import org.jetbrains.exposed.v1.jdbc.*
import org.jetbrains.exposed.v1.jdbc.transactions.transaction

object Users : Table() {
    val id = integer("id").autoIncrement()
    val name = varchar("name", 50)
    override val primaryKey = PrimaryKey(id)
}

fun main() {
    Database.connect("jdbc:h2:mem:test", driver = "org.h2.Driver")

    transaction {
        SchemaUtils.create(Users)
        Users.insert { it[name] = "Andrey" }

        Users.selectAll()
            .where { Users.name eq "Andrey" }
            .forEach { println(it[Users.name]) }
    }
}
```

Every statement must run inside a `transaction { }` block, which resolves the
current connection from a thread-local. There is no ambient connection outside
one.

## Architecture / How It Works

Table schemas are declared as Kotlin `object` singletons — `object Users : Table()`
— where each column is a property carrying its SQL type. Because columns are real
objects, the DSL builds queries by composing them (`Users.name eq "x"`), and the
compiler enforces column/type correctness. The DSL is the foundation; the DAO is
built on top and is optional.

Key internals worth knowing:

- **Thread-local transactions.** `transaction { }` binds a connection to the
  current thread via a `TransactionManager`. Nested `transaction` calls reuse the
  outer transaction by default (savepoints are opt-in). This model is simple but
  is why entities and lazy queries must not escape the block.
- **DAO entities are transaction-scoped.** An `IntEntity` caches its row within
  its transaction; reading a lazy `reference`/`referrersOn` outside that scope
  throws. Relationship access issues a fresh query per entity unless you use
  `.with(...)` eager loading — the standard N+1 source.
- **DSL vs DAO split.** `exposed-core` + `exposed-jdbc` gives you the DSL.
  `exposed-dao` adds entities but is **JDBC-only** — it does not work with the
  R2DBC module.
- **Dialect abstraction.** A per-database dialect layer emits engine-specific SQL,
  supporting H2, PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, and SQLite. The
  project's cuttlefish mascot is the stated metaphor: mimic many engines from one
  API. Coverage is uneven — advanced features (upsert, JSON, window functions)
  are better supported on some dialects than others.
- **Modular by design.** Date/time handling, JSON, money, crypto, Spring
  integration, and migrations are separate artifacts (`exposed-java-time`,
  `exposed-kotlin-datetime`, `exposed-json`, `exposed-spring-boot-starter`, …)
  rather than baked into core.

## Production Notes

- **Schema management is not migrations.** `SchemaUtils.create` and
  `createMissingTablesAndColumns` are convenient for dev but are not a safe
  migration tool — they never drop or alter destructively and can drift from
  reality. Use the newer `exposed-migration-*` modules or an external tool
  (Flyway, Liquibase) for production DDL. Do not point `create*` at a prod
  database.
- **Connection pooling is your job.** `Database.connect` with a raw JDBC URL
  opens connections eagerly. In production, pass a pooled `DataSource`
  (HikariCP). Getting this wrong is the most common performance complaint.
- **N+1 and lazy loading.** The DAO makes it trivial to write a loop that fires
  one query per row. Use `.with()` / `.load()` for eager loading, or drop to the
  DSL for read-heavy paths.
- **Coroutines need the right entry point.** Blocking `transaction { }` ties up a
  thread; use `newSuspendedTransaction` (JDBC) or the R2DBC module for
  non-blocking access. Do not call blocking `transaction` from inside a coroutine
  on a shared dispatcher.
- **The 1.0 upgrade is a real migration.** Moving from `0.x` to 1.0 changes the
  package root to `org.jetbrains.exposed.v1.*` and touches many imports and some
  APIs. Budget for it; follow the official migration guide[^2] rather than
  find-and-replacing blindly.
- **Errors surface late.** Because much is expressed as runtime DSL, some
  mistakes (missing column, bad dialect feature) fail at execution rather than
  compile time. Test against your real target database, not just H2.

## When to Use / When Not

**Use when:**
- You want typesafe SQL in Kotlin without a heavyweight ORM, and prefer writing
  queries close to the SQL they emit.
- You're on the JVM/Kotlin stack and want JetBrains-maintained tooling with
  Spring Boot integration available.
- You need to target multiple SQL engines from one codebase.
- You want to start DSL-only and adopt entities selectively.

**Avoid when:**
- You want full JPA semantics (managed lifecycle, dirty checking, cascades) —
  Hibernate does that; Exposed intentionally does not.
- You want compile-time-verified SQL generated from real schema/queries —
  SQLDelight or jOOQ verify more at build time.
- Your team will lean on the DAO but is unprepared for transaction-scoped
  entities and manual N+1 management.
- You need Kotlin Multiplatform (non-JVM) targets — Exposed is JVM-only.

## Alternatives

- ktorm/ktorm — Kotlin SQL framework with a similar DSL but no transaction-scoped
  entity gotchas; use when you want the query-builder feel without Exposed's DAO
  quirks.
- cashapp/sqldelight — generates typesafe Kotlin from hand-written `.sql`; use
  when you want compile-time-checked SQL and Kotlin Multiplatform support.
- jOOQ/jOOQ — Java, code-generated from your live schema; use when you want the
  most complete, most rigorously typed SQL DSL and JVM-only is fine.
- hibernate/hibernate-orm — full JPA ORM; use when you actually want managed
  entities, dirty tracking, and cascades.
- komapper/komapper — Kotlin ORM with both JDBC and R2DBC via annotation
  processing; use when you want compile-time codegen over Exposed's runtime DSL.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-07 | First commit on GitHub; predates Kotlin 1.0[^1]. |
| 0.x | 2016–2025 | Long-running series; DSL + DAO, dialect support, Spring Boot starter, date-time/JSON/money modules added incrementally. |
| 1.0.0 | 2025+ | Repackage to `org.jetbrains.exposed.v1.*`; R2DBC support added alongside JDBC; migration guide published[^2]. |

## References

[^1]: JetBrains/Exposed repository, created 2013-07-30. https://github.com/JetBrains/Exposed
[^2]: Exposed 1.0.0 migration guide. https://www.jetbrains.com/help/exposed/migration-guide-1-0-0.html
[^3]: Exposed issue tracker (YouTrack, project EXPOSED). https://youtrack.jetbrains.com/issues/EXPOSED

## Tags

kotlin, sql, orm, dao, jvm, jdbc, r2dbc, database, query-builder, jetbrains, type-safe
