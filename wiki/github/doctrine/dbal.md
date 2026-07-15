# doctrine/dbal

> A PHP database abstraction layer over PDO-style drivers, with a portable query API, a typed value-mapping system, and schema introspection/diffing across seven RDBMS engines.

[GitHub repo](https://github.com/doctrine/dbal) ·
[Official website](https://www.doctrine-project.org/projects/dbal.html) ·
[License: MIT](https://github.com/doctrine/dbal/blob/4.4.x/LICENSE)

## Overview

Doctrine DBAL is the layer that sits directly above the raw PHP database drivers (`pdo_mysql`, `pdo_pgsql`, `pdo_sqlite`, `oci8`, `pdo_sqlsrv`, `ibm_db2`, and the mysqli path) and directly below Doctrine ORM[^1]. It provides a uniform connection and query API, a type system that converts between PHP and database representations, and — its most distinctive component — a schema introspection and comparison engine that can read an existing database's structure and compute the DDL needed to migrate one schema to another.

The project has existed as part of Doctrine since 2010 but is deliberately narrow: it is *not* an ORM. There is no object mapping, no unit of work, no lazy loading. You get connections, a fluent `QueryBuilder` that emits SQL (not DQL), prepared-statement helpers, transactions, and platform-aware schema tooling. Most PHP developers meet DBAL indirectly — it is the foundation of Doctrine ORM, doctrine/migrations, and a large fraction of Symfony's database layer — rather than depending on it by name[^2].

The defining tension is **portability versus fidelity**. DBAL abstracts eight database dialects behind one API, but SQL dialects, type systems, and especially schema metadata differ in ways no abstraction fully hides. The schema comparator, in particular, must reverse-engineer intent from what each driver reports about columns, indexes, and foreign keys — and different engines report different things. This is the source of both the library's value and its most persistent operational friction.

## Getting Started

```bash
composer require doctrine/dbal
```

```php
<?php
use Doctrine\DBAL\DriverManager;

$conn = DriverManager::getConnection([
    'dbname'   => 'app',
    'user'     => 'app',
    'password' => 'secret',
    'host'     => '127.0.0.1',
    'driver'   => 'pdo_mysql',
]);

// Parameterized query — placeholders are bound, never interpolated.
$rows = $conn->fetchAllAssociative(
    'SELECT id, email FROM users WHERE active = ?',
    [1]
);

// Fluent QueryBuilder — emits SQL, not ORM DQL.
$qb = $conn->createQueryBuilder();
$qb->select('u.id', 'u.email')
   ->from('users', 'u')
   ->where('u.created_at > :since')
   ->setParameter('since', '2026-01-01');

$recent = $qb->executeQuery()->fetchAllAssociative();
```

## Architecture / How It Works

DBAL is organized as a stack of replaceable middleware and platform-specific strategy objects:

- **Driver / Connection / Result.** A thin wrapper over the underlying driver. Since DBAL 3 the result API is explicit about shape: `fetchAssociative()`, `fetchNumeric()`, `fetchOne()`, and their `fetchAll*` / `iterate*` variants replaced the old polymorphic `fetch()` / `fetchAll()` that took a fetch-mode flag[^3]. `Connection` no longer extends PDO.
- **Driver middleware.** Cross-cutting concerns (logging via PSR-3, query timing) are implemented as middleware wrapping the driver rather than as connection subclasses — a structural change introduced in DBAL 3.
- **Platforms.** `AbstractPlatform` and its subclasses (`MySQLPlatform`, `PostgreSQLPlatform`, `SQLServerPlatform`, `OraclePlatform`, `SQLitePlatform`, `DB2Platform`, plus version-specific variants) encode each engine's SQL grammar: how to quote identifiers, generate `LIMIT`, express column types, and emit DDL.
- **Types.** The `Types` registry maps a logical DBAL type (`string`, `json`, `datetime_immutable`, `guid`, …) to a platform SQL declaration and to `convertToPHPValue` / `convertToDatabaseValue` hooks. Custom types are registered globally on `Type`.
- **Schema.** `AbstractSchemaManager` introspects a live database into `Schema` / `Table` / `Column` objects. `Comparator` diffs two `Schema` instances into a `SchemaDiff`, which a platform turns into `ALTER` statements. doctrine/migrations is built entirely on this diffing machinery.

The QueryBuilder is a string-assembly tool, not a query planner — it does not validate columns, know about relations, or protect against generating invalid SQL for a given platform. It exists to make dynamic SQL composition and parameter binding safer, not to hide SQL.

## Production Notes

**Schema-comparator false diffs are the number-one footgun.** Because the comparator infers column definitions from what each driver reports, small representational mismatches — integer display widths on MySQL, default-value quoting, `BOOLEAN` vs `TINYINT(1)`, unsigned flags, collation/charset defaults, comment-encoded type hints — routinely produce "phantom" diffs where `doctrine:migrations:diff` generates an `ALTER TABLE` that changes nothing meaningful. Teams learn to review every generated migration by hand and never trust an auto-diff blindly.

**Historically, DBAL smuggled type metadata into column comments** (e.g. `(DC2Type:json)`) so it could round-trip types the database had no native equivalent for. This leaks Doctrine implementation detail into your schema and was a recurring source of comparator noise; native `json` and other types have reduced but not eliminated the pattern. Inspect column comments before assuming a diff is spurious.

**The DBAL 2 → 3 upgrade is a genuine rewrite, not a point bump.** The result/fetch API changed shape, `Connection` stopped extending PDO, fetch-mode constants were removed, and numerous method signatures changed[^3]. DBAL 3 → 4 then removed a large batch of APIs deprecated across the 3.x line[^4]. Both jumps typically require code changes in any project that touched DBAL directly rather than only through the ORM. Pin the major version and read the UPGRADE notes per step.

**Performance.** The abstraction is thin; per-query overhead over raw PDO is small and rarely the bottleneck. Schema introspection is the expensive operation — reading a large database's full structure issues many metadata queries — so avoid running `SchemaManager` introspection on hot paths; it belongs in migration/CI tooling, not request handling.

**Driver coverage is uneven.** MySQL/MariaDB, PostgreSQL, and SQLite are exercised heavily and behave predictably. SQL Server, Oracle, and IBM DB2 are supported and CI-tested but see far less real-world traffic; expect rougher edges in schema introspection and less community precedent for corner cases on those platforms.

## When to Use / When Not

**Use when:**
- You need to run the same application against more than one RDBMS, or want to keep that option open.
- You want typed value conversion and safe parameter binding without adopting a full ORM.
- You are building migration or schema-management tooling and need programmatic schema diffing.
- You are already in the Doctrine/Symfony ecosystem (you likely depend on it transitively anyway).

**Avoid when:**
- You target exactly one database forever and want zero abstraction — raw PDO or the native driver is simpler and has no version-migration tax.
- You want ActiveRecord ergonomics or object mapping — that is the ORM's job, not DBAL's.
- You rely on database-specific features (advanced Postgres types, stored-proc-heavy designs) that the portable layer obscures more than it helps.

## Alternatives

- doctrine/orm — use when you want full data-mapper object mapping; it sits on top of DBAL, so this is a layer choice, not a replacement.
- illuminate/database — use when you are in Laravel and want Eloquent/ActiveRecord plus a query builder in one package.
- cycle/orm — use when you want a data-mapper ORM (Spiral ecosystem) with schema declaration outside of annotations.
- cakephp/database — use when you want a standalone query builder + schema layer with a lighter footprint than DBAL.
- PDO (PHP core) — use when you need no portability or type system and want the thinnest possible driver access.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.0 | 2011 | First long-lived major line; PDO-centric, fetch-mode result API. |
| 3.0 | 2020 | Rewrite: explicit `fetch*Associative/Numeric` API, driver middleware, `Connection` no longer extends PDO[^3]. |
| 4.0 | 2024 | Removal of APIs deprecated across 3.x; stricter types, dropped legacy platform/driver code[^4]. |
| 4.4 | 2026 | Current stable line (default branch `4.4.x`). |
| 5.0 | dev | In development on `5.0.x`. |

## References

[^1]: Doctrine DBAL documentation — Introduction. https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/introduction.html
[^2]: Doctrine Project — DBAL overview. https://www.doctrine-project.org/projects/dbal.html
[^3]: Doctrine DBAL 3 UPGRADE notes (result API, PDO decoupling, middleware). https://github.com/doctrine/dbal/blob/4.4.x/UPGRADE.md
[^4]: Doctrine DBAL 4 UPGRADE notes (removal of deprecated 3.x APIs). https://github.com/doctrine/dbal/blob/4.4.x/UPGRADE.md

## Tags

php, database, orm, sql, database-abstraction, schema-migration, mysql, postgresql, sqlite, doctrine, query-builder
