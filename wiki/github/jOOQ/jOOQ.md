# jOOQ/jOOQ

> A type-safe internal DSL and code generator that models SQL as a Java API, so queries are checked by the Java compiler against your real schema.

[GitHub repo](https://github.com/jOOQ/jOOQ) ·
[Official website](https://www.jooq.org) ·
License: dual — Apache-2.0 (Open Source Edition) / commercial (see below)

## Overview

jOOQ ("Java Object Oriented Querying") is a database-first library that inverts the usual ORM premise: instead of hiding SQL behind objects, it makes SQL itself first-class and type-safe in Java. A code generator reads your database schema and emits Java classes for every table, column, key, and routine; you then write queries with a fluent DSL (`select(...).from(...).where(...)`) that mirrors SQL syntax closely enough to read like SQL, while the Java compiler rejects typos, type mismatches, and references to columns that do not exist[^1]. It is authored and maintained primarily by Lukas Eder through Data Geekery GmbH (Switzerland)[^2].

The defining tension is **code generation vs. dynamism**. jOOQ's headline safety guarantee — the compiler knows your schema — only holds if you run the generator against a real schema (a live dev database, a migration script, or an XML/DDL definition) as a build step. That coupling to a concrete schema is exactly what buys the type safety, and exactly what makes jOOQ heavier to bootstrap than a string-based query tool. Teams that want dynamic, schema-agnostic queries can use jOOQ without generation, but then they give up most of the reason to choose it.

The second defining tension is **licensing**. jOOQ is dual-licensed: the Open Source Edition (Apache-2.0) supports open-source databases (PostgreSQL, MySQL, MariaDB, SQLite, and others), while support for commercial databases (Oracle, SQL Server, DB2, and similar) requires a paid commercial edition[^3]. GitHub reports the license as `NOASSERTION` because the repository combines the ASL2.0 open-source tree with commercially-licensed distributions. This is unusual for a widely-used Java library and is the single most common source of surprise for new adopters.

## Getting Started

Maven dependency (Open Source Edition):

```xml
<dependency>
  <groupId>org.jooq</groupId>
  <artifactId>jooq</artifactId>
  <version>3.19.0</version>
</dependency>
```

A query, once the generator has produced `Tables.AUTHOR` / `Tables.BOOK` from your schema:

```java
import static org.jooq.impl.DSL.*;
import static com.example.generated.Tables.*;

try (Connection conn = DriverManager.getConnection(url, user, pass)) {
    DSLContext ctx = DSL.using(conn, SQLDialect.POSTGRES);

    Result<Record2<String, Integer>> result =
        ctx.select(AUTHOR.LAST_NAME, count())
           .from(AUTHOR)
           .join(BOOK).on(BOOK.AUTHOR_ID.eq(AUTHOR.ID))
           .where(BOOK.PUBLISHED_YEAR.gt(2000))
           .groupBy(AUTHOR.LAST_NAME)
           .fetch();

    for (Record2<String, Integer> r : result)
        System.out.println(r.value1() + ": " + r.value2());
}
```

Change `LAST_NAME` to a column that does not exist, or compare it to an `Integer`, and the code does not compile.

## Architecture / How It Works

jOOQ has three layers that are usually conflated but are separable:

1. **The code generator** — a build-time tool (Maven/Gradle plugin, CLI, or programmatic) that connects to a schema source and emits Java classes: `Table`, `TableRecord`/`UpdatableRecord`, `Key`, `Sequence`, and routine bindings. The schema source can be a live JDBC connection, an XML dump, or DDL scripts parsed by jOOQ's own SQL parser (the `DDLDatabase`)[^4]. The generated code is the type-safe surface everything else builds on.
2. **The DSL / query model** — a fluent builder that constructs an internal AST of `QueryPart` objects. Nothing executes at build time; `select(...).from(...)` returns immutable query objects. This model API can be traversed and rewritten, which powers jOOQ's SQL transformation and parsing features[^5].
3. **The rendering + execution layer** — a `DSLContext` bound to a `SQLDialect` renders the AST to dialect-specific SQL and runs it over JDBC (or R2DBC for reactive). The same query object renders differently per dialect; jOOQ emulates missing features (e.g. `LIMIT` on Oracle, `MULTISET` via SQL/JSON where not native).

**Dialect emulation** is the deep value. jOOQ knows 30+ SQL dialects and rewrites standard constructs into each. The `MULTISET` operator (3.15, 2021) is the clearest example: it lets you nest collections directly in a query and map them to Java records in one type-safe round trip, emulated with `jsonb_agg` / SQL-XML on databases that lack native `MULTISET`[^6].

**It is not an ORM in the JPA sense.** There is no persistence context, no dirty-tracking session, no lazy-loading proxy graph. `UpdatableRecord` provides active-record-style `store()`/`delete()` CRUD, and DAOs offer light repository methods, but there is no first-level cache and no entity manager. Objects do not diverge from SQL — that is the point.

## Production Notes

**The generator is a build dependency on a schema.** CI must be able to reach a schema source. The three practical patterns: (a) point the generator at a throwaway database that migrations (Flyway/Liquibase) build fresh; (b) generate from DDL scripts via `DDLDatabase` with no live DB; (c) generate once and commit the output. Committing generated code avoids the CI database but means the code and schema can silently drift. Most mature setups use (a) or (b) so the generated code is never hand-editable and always matches migrations.

**Licensing is a runtime and legal footgun, not just a purchase decision.** The Open Source Edition targets the *latest* versions of open-source databases; support for older database versions and for all commercial databases lives in the commercial editions[^3]. A team on Oracle or SQL Server cannot use the free edition in production at all. Confirm your database and version are covered before committing architecturally.

**Generated code volume.** Large schemas (hundreds of tables) produce a lot of generated Java, which lengthens compile times and can inflate IDE indexing. Generation config (`includes`/`excludes`, per-schema generation, `generatedSerialVersionUID`) matters at scale.

**Dialect leakage.** Because jOOQ renders per dialect, code written and tested against one database can behave differently on another even though it compiles. Emulations are faithful but not always identical in edge cases (null ordering, empty-`MULTISET` handling, type coercion). Integration-test against the actual target database, not just an in-memory stand-in.

**Java version coupling.** jOOQ distributions are tied to Java baselines, and the open-source edition tracks a relatively recent JDK; older-JDK support has historically been part of the commercial offering. Verify the JDK requirement of the specific jOOQ version against your runtime[^7].

**Reactive is real but narrower.** R2DBC support exists (fetching, some execution), but the API surface and driver maturity lag the blocking JDBC path. Treat reactive jOOQ as supported-but-younger.

## When to Use / When Not

**Use when:**
- SQL is central to the product and you want it compiler-checked, not string-typed.
- You target PostgreSQL/MySQL/MariaDB (free edition) or already license a commercial DB and can pay for jOOQ.
- You want advanced SQL (window functions, CTEs, `MULTISET`, procedural logic) rendered portably across dialects.
- You want CRUD without a full ORM's session/caching machinery.

**Avoid when:**
- You cannot run a code-generation step against a schema in your build.
- You need a JPA-style persistence context, entity graph, and dirty tracking (use Hibernate/JPA).
- You're on a commercial database but cannot license the commercial edition.
- Your access is simple, dynamic, and schema-agnostic — a thin JDBC helper is lighter.

## Alternatives

- hibernate/hibernate-orm — full JPA ORM with a persistence context; use when you want entity-graph mapping and caching rather than SQL-first control.
- mybatis/mybatis-3 — SQL-in-XML/annotations mapper; use when you want to hand-write SQL without a code-generation step.
- ktorm/ktorm — Kotlin-native type-safe SQL DSL; use when you're all-Kotlin and want an idiomatic DSL over jOOQ's Java-first API.
- querydsl/querydsl — type-safe query DSL spanning JPA/SQL; use when you need one query API over both JPA entities and raw SQL.
- exposed (JetBrains) — Kotlin SQL DSL and light DAO; use for smaller Kotlin services that don't need jOOQ's dialect breadth.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | ~2009 | Initial releases by Lukas Eder; early DSL + generator[^2]. |
| 3.0 | 2013 | Major API redesign; Data Geekery GmbH founded, commercial editions introduced[^2]. |
| 3.11 | 2018 | SQL parser and translator maturation. |
| 3.15 | 2021-07 | `MULTISET`/`ROW` nested collections; R2DBC reactive fetching[^6]. |
| 3.17 | 2022 | Continued dialect and code-generation expansion. |
| 3.19 | 2023-12 | Ongoing dialect support, parser, and generator improvements[^8]. |

## References

[^1]: jOOQ README and manual, "DSL API for type-safe query construction." https://www.jooq.org/doc/latest/manual/sql-building/dsl-api/
[^2]: Data Geekery GmbH, jOOQ vendor/company. https://www.jooq.org/legal/licensing
[^3]: jOOQ licensing and database support matrix. https://www.jooq.org/download/#databases
[^4]: jOOQ manual, code generation configuration (`DDLDatabase`, XML/DDL sources). https://www.jooq.org/doc/latest/manual/code-generation/
[^5]: jOOQ manual, Model API for traversal and replacement. https://www.jooq.org/doc/latest/manual/sql-building/model-api/
[^6]: Data Geekery, "jOOQ 3.15's new MULTISET operator." https://blog.jooq.org/jooq-3-15s-new-multiset-operator-will-change-how-you-think-about-sql/
[^7]: jOOQ manual, versions and Java baselines. https://www.jooq.org/doc/latest/manual/getting-started/
[^8]: jOOQ release notes. https://www.jooq.org/notes/

## Tags

java, sql, jdbc, query-builder, code-generation, type-safe, database, orm-alternative, r2dbc, dsl
