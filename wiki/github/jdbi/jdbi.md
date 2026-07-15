# jdbi/jdbi

> A thin, idiomatic convenience layer over JDBC for the JVM — you still write the SQL, Jdbi handles the boilerplate and the mapping.

[GitHub repo](https://github.com/jdbi/jdbi) ·
[Official website](http://jdbi.org/) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0.html)

## Overview

Jdbi (originally JDBI) is a database access library for Java and other JVM
languages, built directly on top of JDBC[^1]. It deliberately occupies the
middle ground between raw JDBC — verbose, error-prone, resource-leak-prone —
and full object-relational mappers like Hibernate. Jdbi is not an ORM: there is
no entity lifecycle, no identity map, no lazy-loading proxies, and no
dialect-generating query language. You write SQL strings; Jdbi binds
parameters, executes, and maps result rows to objects. If your database has a
JDBC driver, Jdbi works with it.

The library exposes two API styles that share one engine. The **Core / Fluent
API** is imperative: you open a `Handle` (a wrapped `Connection`), call
`createQuery`, bind arguments, and choose a mapper. The **SQL Object API** is
declarative: you annotate a Java interface with `@SqlQuery` / `@SqlUpdate`
strings, and Jdbi generates the implementation at runtime[^2]. Most codebases
mix both — Fluent for ad-hoc and dynamic queries, SQL Object for the stable DAO
surface.

The founder is Brian McCallister (@brianm); the project dates to 2009 and the
current maintained line is Jdbi 3, whose packages live under `org.jdbi`[^3].
Jdbi 2 (the old `org.skife.jdbi` namespace) is end-of-life and shares almost no
code with 3.x — treat them as different libraries. Jdbi's defining tension is
that it gives you convenience without hiding SQL: this is liberating if you know
your schema and painful if you wanted the framework to author queries for you.

## Getting Started

```xml
<!-- Maven — core plus the SQL Object extension -->
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-core</artifactId>
  <version>3.x.y</version>
</dependency>
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-sqlobject</artifactId>
  <version>3.x.y</version>
</dependency>
```

```java
// Fluent API — install plugins explicitly, then open a Handle
Jdbi jdbi = Jdbi.create("jdbc:postgresql://localhost/app", "user", "pass");
jdbi.installPlugin(new SqlObjectPlugin());

List<User> users = jdbi.withHandle(handle ->
    handle.createQuery("SELECT id, name FROM users WHERE active = :active")
          .bind("active", true)
          .mapToBean(User.class)
          .list());
```

```java
// SQL Object API — declarative DAO interface
public interface UserDao {
    @SqlQuery("SELECT id, name FROM users WHERE id = :id")
    @RegisterBeanMapper(User.class)
    User findById(@Bind("id") long id);

    @SqlUpdate("INSERT INTO users (name) VALUES (:name)")
    @GetGeneratedKeys
    long insert(@Bind("name") String name);
}

jdbi.useExtension(UserDao.class, dao -> dao.insert("Ada"));
```

## Architecture / How It Works

The object graph is small. A **`Jdbi`** instance wraps a `DataSource` or
connection factory and holds shared configuration (registered mappers,
arguments, plugins). A **`Handle`** wraps a single JDBC `Connection` and is the
unit of transaction; it is not thread-safe and must be closed. The
`withHandle` / `useHandle` and `inTransaction` / `useTransaction` callback forms
exist so that Jdbi owns the try-with-resources lifecycle for you — the most
common Jdbi bug is holding a `Handle` open across threads or leaking one by
using the manual `open()` form without closing.

Mapping is built from three registries. **Argument** binders turn Java values
into JDBC parameters; **ColumnMapper** turns one column into a value;
**RowMapper** turns a `ResultSet` row into an object. Higher-level helpers
(`mapToBean`, `@RegisterConstructorMapper`, `mapToMap`) are built on these.
Everything is a plugin or a registration: Jdbi core ships almost nothing for
third-party types, and you pull capability in — `jdbi3-postgres` for arrays /
`hstore` / `uuid`, `jdbi3-kotlin` for data classes and named parameters from
constructor names, `jdbi3-guava`, `jdbi3-jackson2`, `jdbi3-immutables`, and so
on. Forgetting to `installPlugin` is the second most common source of confusion,
because the failure surfaces as a generic "no mapper registered" at runtime.

The SQL Object API is a runtime proxy: annotated interface methods are
dispatched through handlers that read the `@Sql*` annotation, bind parameters by
name, and invoke the same Core machinery. Named parameters (`:name`) are parsed
by Jdbi itself, not by JDBC — Jdbi rewrites the statement into positional `?`
placeholders before execution, which is also how it supports list expansion via
`bindList`. Template engines (`DefinedAttributeTemplateEngine`, or the optional
`jdbi3-freemarker` / StringTemplate integrations) handle the cases where the SQL
text itself must vary.

## Production Notes

- **`Handle` is a live connection.** Keep them short-lived. Long-held handles
  pin a pooled connection and starve the pool; handles crossing threads cause
  intermittent, hard-to-reproduce failures. Prefer the callback forms
  (`withHandle`, `inTransaction`) over manual `open()`/`close()`.
- **Connection pooling is your job.** Jdbi does not pool; it wraps a
  `DataSource`. In production you point it at HikariCP or equivalent. A bare
  `Jdbi.create(url, user, pass)` opens a new connection per handle and is fine
  only for scripts and tests.
- **Plugins must be installed before use.** Kotlin data classes, Postgres
  arrays, and Guava/Immutables/Jackson types all require their plugin. Missing
  registration fails at query time, not startup — add an integration test that
  exercises each mapper.
- **Named-parameter parsing is a footgun with literal colons.** Because Jdbi
  parses `:name` itself, PostgreSQL `::type` casts and colons inside string
  literals can be misread. Use the `?` positional form or the doubled/escaped
  syntax where the parser fights you.
- **No N+1 protection.** Unlike an ORM, Jdbi will not batch or cache for you.
  `reduceRows` / `@UseRowReducer` exist to fold one-to-many joins into object
  graphs, but you must design the query and reducer deliberately.
- **Java baseline moves.** Jdbi 3.40.0 dropped Java 8/9/10; 3.50.0 requires
  Java 17 or newer[^4]. Pinning an old 3.x to stay on an old JDK is viable but
  freezes you out of fixes. Public API follows SemVer, so within a major line
  upgrades are low-risk.

## When to Use / When Not

**Use when:**
- You want to write SQL by hand and keep full control of the query, but not the
  JDBC boilerplate, resource handling, or result-set-to-object mapping.
- You are on Postgres or another SQL database and value predictable, inspectable
  queries over generated ones.
- You use Kotlin, Guava, Immutables, or Vavr types and want first-class mapping
  via a plugin.
- You want a small dependency footprint and no bytecode weaving or startup
  entity scanning.

**Avoid when:**
- You want the framework to generate SQL and manage entity state — reach for a
  JPA implementation instead.
- You want compile-time-checked, type-safe query construction — jOOQ or Exposed
  fit better.
- Your team expects an identity map, dirty checking, cascade persistence, or
  lazy associations; Jdbi has none of these by design.

## Alternatives

- hibernate/hibernate-orm — use instead when you want a full JPA ORM with entity
  lifecycle, caching, and generated SQL rather than hand-written queries.
- jOOQ/jOOQ — use instead when you want type-safe, compile-checked SQL built from
  a generated schema model.
- mybatis/mybatis-3 — use instead when you prefer externalized XML/annotation SQL
  mapping with a mature mapper ecosystem.
- spring-projects/spring-framework (`JdbcClient`/`JdbcTemplate`) — use instead
  when you are already all-in on Spring and want its native JDBC helper.
- JetBrains/Exposed — use instead when you are Kotlin-first and want a Kotlin DSL
  or lightweight DAO layer rather than raw SQL strings.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2009 | Project created by Brian McCallister; Jdbi 2.x under `org.skife.jdbi`[^3]. |
| 3.0 | 2017-09 | Ground-up rewrite; `org.jdbi` packages, plugin architecture, SQL Object redesign[^2]. |
| 3.40.0 | 2023 | Dropped Java 8/9/10 support[^4]. |
| 3.50.0 | 2025 | Raised baseline to Java 17; CI tested on 17/21/25[^4]. |

## References

[^1]: Jdbi project site — Developer Guide. https://jdbi.org/
[^2]: Jdbi 3 Developer Guide — SQL Objects. https://jdbi.org/#_sql_objects
[^3]: Jdbi GitHub repository (founder, project members, license). https://github.com/jdbi/jdbi
[^4]: Jdbi README — Prerequisites and Java version compatibility. https://github.com/jdbi/jdbi/blob/master/README.md

## Tags

java, jvm, jdbc, sql, database, kotlin, data-access, dao, sql-mapper, postgresql, apache-2.0
