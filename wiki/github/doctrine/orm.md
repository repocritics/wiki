# doctrine/orm

> A Data Mapper ORM for PHP — persistence for plain objects, with its own query language and a Unit of Work that batches every change into one flush.

[GitHub repo](https://github.com/doctrine/orm) ·
[Official website](https://www.doctrine-project.org/projects/orm.html) ·
[License: MIT](https://github.com/doctrine/orm/blob/3.6.x/LICENSE)

## Overview

Doctrine ORM is an object-relational mapper for PHP 8.1+ that persists ordinary PHP objects without requiring them to extend a base class or know about the database[^1]. It implements the Data Mapper pattern, the deliberate opposite of the Active Record pattern used by Laravel's Eloquent: entities are plain objects, and a separate `EntityManager` handles loading, tracking, and saving them. This separation is the source of both its reputation for "enterprise" correctness and its reputation for being heavy.

The two defining pieces are the **Unit of Work** and **DQL**. The Unit of Work keeps an in-memory identity map of every managed entity, tracks changes to them, and computes the minimal set of INSERT/UPDATE/DELETE statements only when you call `flush()`. DQL (Doctrine Query Language) is an object-oriented query dialect modeled on Hibernate's HQL — you query entities and their mapped relations rather than tables and columns[^1]. Doctrine ORM sits on top of Doctrine DBAL, a separate database abstraction and query-builder package that handles connections, platforms, and type conversion.

Doctrine is the default ORM in the Symfony ecosystem (via `doctrine-bundle`) and the persistence layer under API Platform, which is where most of its production usage concentrates. The central tension: its object-graph model and lazy-loading proxies make rich domain models pleasant to write, but the same machinery makes performance and memory behavior non-obvious, and it punishes code that treats it like a thin query builder.

## Getting Started

```bash
composer require doctrine/orm doctrine/dbal
```

```php
<?php
// src/Entity/User.php — mapping via PHP 8 attributes
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'users')]
class User
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    public int $id;

    #[ORM\Column(length: 255)]
    public string $name;
}
```

```php
<?php
use Doctrine\ORM\ORMSetup;
use Doctrine\ORM\EntityManager;
use Doctrine\DBAL\DriverManager;

$config = ORMSetup::createAttributeMetadataConfiguration([__DIR__ . '/src'], isDevMode: true);
$connection = DriverManager::getConnection(['url' => 'sqlite:///db.sqlite'], $config);
$em = new EntityManager($connection, $config);

$user = new User();
$user->name = 'Tom';
$em->persist($user);   // stage for insertion
$em->flush();          // one transaction, all pending changes

$found = $em->getRepository(User::class)->findOneBy(['name' => 'Tom']);
```

## Architecture / How It Works

The `EntityManager` is the façade over the whole system; almost every interaction goes through it. Behind it sit several coupled subsystems:

- **Metadata mapping.** Doctrine reads mapping from PHP 8 attributes or XML (as of 3.x). It builds `ClassMetadata` objects that describe tables, columns, relations, and inheritance. Metadata is expensive to compute, so it is cached (APCu, PSR-6, or files) in production.
- **Unit of Work.** The heart of the ORM. It holds the **identity map** (one PHP object instance per row per manager), snapshots managed entities on load, and on `flush()` runs change-set computation to emit only the SQL that actually changed. It also orders operations by foreign-key dependency and wraps them in a transaction.
- **Hydration.** After DBAL returns rows, hydrators turn them into results: object hydration (full entities), array hydration, and scalar hydration. Object hydration is the default and the most expensive.
- **Proxies.** Lazy associations are filled with generated proxy subclasses that load the real row on first access. This is what makes `$user->getOrders()` "just work" — and what silently triggers N+1 queries.
- **DQL + SQL walkers.** DQL is parsed into an AST and compiled to platform-specific SQL by walker classes. The `QueryBuilder` produces DQL, not SQL. Native SQL is available via `NativeQuery` with a `ResultSetMapping`.
- **Doctrine DBAL** underneath handles the connection, platform differences, and type system. The ORM does not talk to PDO directly.

The coupling story matters: the identity map, proxies, and Unit of Work are one system, not optional layers. You cannot use the change-tracking without paying for the identity map, and lazy loading is on unless you opt out per association or per query.

## Production Notes

**The N+1 problem is the default failure mode.** Iterating a collection of entities and touching a lazy relation issues one query per entity. The fix is explicit `JOIN ... addSelect` (fetch joins) in DQL, or `EAGER` fetch on the association, or the newer fetch-mode overrides. There is no automatic batching; you have to know the relation will be accessed.

**Memory in long-running processes.** The identity map holds a reference to every managed entity for the lifetime of the `EntityManager`, so batch scripts and queue workers (Symfony Messenger, RoadRunner, Swoole) leak memory unless you call `$em->clear()` periodically or process in chunks. For read-heavy batches use `toIterable()` plus `clear()`, array hydration, or the read-only query hint to skip change tracking.

**`flush()` is all-or-nothing by default.** A bare `flush()` computes change sets for every managed entity and commits them in one transaction. In large Units of Work this is costly; batch inserts should flush and clear in windows (e.g. every 100–1000 entities).

**Caching is layered and easy to misconfigure.** Metadata cache and query (DQL→SQL) cache are effectively mandatory in production — without them Doctrine re-parses mapping and DQL on every request. The result cache is opt-in per query. The **second-level cache** (entity/collection caching across requests) exists but is documented as experimental and is not a drop-in; most teams cache at the application layer instead.

**Schema and migrations are separate concerns.** The `SchemaTool` / `orm:schema-tool` can diff and update schema but is not meant for production migrations. `doctrine/migrations` is the separate package used for versioned, reviewable migrations.

**Upgrade pain — 2.x to 3.0.** Doctrine ORM 3.0 removed annotation mapping and YAML mapping entirely; projects had to migrate to attributes or XML first[^2]. It also dropped older PHP versions (requires PHP 8.1+) and tightened types on many entity properties, surfacing latent nullability bugs. The `doctrine/orm` and `doctrine/dbal` version matrices are independent, so a major DBAL bump can arrive separately and break custom types or platform code.

## When to Use / When Not

**Use when:**
- You want a rich domain model with plain PHP entities decoupled from the database (DDD-style aggregates, value objects via embeddables).
- You're on Symfony or API Platform, where Doctrine is the supported, integrated path.
- Your queries span object graphs and you want DQL/QueryBuilder over hand-written SQL joins.
- You need database-portable code across MySQL/PostgreSQL/SQLite/etc. via DBAL's platform layer.

**Avoid when:**
- You want a thin, convention-driven Active Record layer for CRUD apps — Eloquent is simpler and faster to reach for.
- You run persistent async workers and cannot afford the identity-map memory model — Cycle ORM is designed for that shape.
- Your workload is analytics/reporting with large result sets — hydration overhead dominates; use DBAL or raw SQL.
- The team lacks the bandwidth to learn the Unit of Work; misused, Doctrine produces the slow, surprising queries it's blamed for.

## Alternatives

- laravel/framework — Eloquent is an Active Record ORM; use it when you want convention-over-configuration CRUD and don't need domain/persistence separation.
- cycle/orm — a Data Mapper ORM built for long-running PHP (RoadRunner, Swoole); use it when persistent workers make Doctrine's identity-map memory model a problem.
- doctrine/dbal — the abstraction/query-builder layer beneath the ORM; use it directly when you want portable SQL without object mapping or Unit of Work overhead.
- propelorm/Propel — a code-generation ORM combining Active Record with generated query classes; use it when you prefer generated, IDE-friendly model code.
- sqlalchemy/sqlalchemy — the Python analog (Data Mapper + Unit of Work); the closest cross-language reference for the same design philosophy.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.0 | 2010 | Complete rewrite from Doctrine 1.x's Active Record to the Data Mapper + Unit of Work model; introduced DQL and the EntityManager[^1]. |
| 2.5 | 2015 | Embeddables (value objects) and the experimental second-level cache. |
| 2.9 | 2021 | PHP 8 attribute mapping added alongside annotations. |
| 2.14 | 2023 | Late 2.x line; annotations deprecated ahead of 3.0. |
| 3.0 | 2024 | Removed annotation and YAML mapping; PHP 8.1+ only; stricter entity types[^2]. |
| 3.x | 2024–2026 | Current stable line (default branch 3.6.x); ongoing 3.x releases[^3]. |
| 4.0 | in development | Active 4.0.x branch; next major line. |

## References

[^1]: Doctrine ORM README and documentation — object-relational mapper for PHP 8.1+, Data Mapper pattern, DQL inspired by Hibernate HQL. https://www.doctrine-project.org/projects/doctrine-orm/en/stable/index.html
[^2]: Doctrine ORM "Upgrade to 3.0" documentation — removal of annotation/YAML drivers and PHP 8.1+ requirement. https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/changelog.html
[^3]: doctrine/orm repository, default branch 3.6.x, last pushed 2026-07-13. https://github.com/doctrine/orm

## Tags

php, orm, data-mapper, persistence, database, sql, dql, symfony, doctrine, unit-of-work, entity-manager
