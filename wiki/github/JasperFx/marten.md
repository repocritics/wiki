# JasperFx/marten

> A .NET library that turns PostgreSQL into both a document database and an ACID event store, using JSONB storage and code-generated data access.

[GitHub repo](https://github.com/JasperFx/marten) ·
[Official website](https://martendb.io) ·
[License: MIT](https://github.com/JasperFx/marten/blob/master/LICENSE)

## Overview

Marten is a .NET persistence library that treats a single PostgreSQL database as
two things at once: a document database (arbitrary C# objects serialized to
`jsonb` columns, queried through a LINQ provider) and an append-only event store
with user-defined projections[^1]. It started in 2015 as an attempt to replace
RavenDB at a client shop while keeping Postgres as the storage engine, and grew
into the flagship of the "Critter Stack" alongside the Wolverine messaging
library[^2]. The lead maintainers are Jeremy D. Miller, Babu Annamalai, Oskar
Dudycz, and Joona-Pekka Kokko, with commercial support sold through JasperFx
Software[^3].

The defining bet is that a relational database most teams already run can serve
document and event-sourcing workloads well enough to avoid adopting MongoDB,
Cosmos DB, or a dedicated event store — while keeping real transactions across
documents and events in one `SaveChangesAsync`. The cost of that bet is that you
inherit Postgres's constraints (single-node write scaling, JSONB query
ergonomics) and Marten's own machinery: code generation, an async projection
daemon, and a schema-management layer that will happily rewrite your database if
you let it. It is a comfortable fit for .NET-on-Postgres shops and an awkward one
for anyone who wanted a distributed document store.

## Getting Started

```bash
dotnet add package Marten
```

```csharp
using Marten;

// Register with the .NET generic host / DI container
builder.Services.AddMarten(opts =>
{
    opts.Connection(builder.Configuration.GetConnectionString("marten")!);
    // In dev only — auto-apply schema changes. Do NOT use in production.
    opts.AutoCreateSchemaObjects = AutoCreate.CreateOrUpdate;
});

// Document usage
public record User(Guid Id, string Name, string[] Roles);

await using var session = store.LightweightSession();
session.Store(new User(Guid.NewGuid(), "Jane", new[] { "admin" }));
await session.SaveChangesAsync();

var admins = await session.Query<User>()
    .Where(u => u.Roles.Contains("admin"))
    .ToListAsync();
```

```csharp
// Event sourcing
var streamId = session.Events.StartStream<Order>(
    new OrderCreated(customerId), new OrderShipped()).Id;
await session.SaveChangesAsync();

var order = await session.Events.AggregateStreamAsync<Order>(streamId);
```

## Architecture / How It Works

Marten sits on top of **Npgsql** and generates SQL directly rather than going
through an ORM. Each document type maps to one table holding a `jsonb` `data`
column plus system columns (id, version, timestamps, optional tenant id).
Serialization is pluggable between **System.Text.Json** and **Newtonsoft.Json**;
the choice is durable, because it decides how existing rows deserialize[^4].

The document side is a unit-of-work (`IDocumentSession`) with change tracking,
optimistic/exclusive concurrency, computed and duplicated indexes (values lifted
out of JSONB into real columns so Postgres can index them), and a LINQ provider
that compiles expression trees into JSONB path queries. `Include()` supports a
limited form of related-document loading, but the provider is not a full ORM:
some LINQ shapes are unsupported and throw at query time rather than falling back
silently.

The event store is a separate feature set sharing the same session and
transaction. Events append to a stream; **projections** derive read models from
those streams and run in one of three modes: *inline* (updated in the same
transaction as the append), *live* (aggregated on read, nothing persisted), or
*async* (built in the background by the **async daemon**)[^5]. The async daemon
uses Postgres advisory locks for leader election so that only one node processes
a given projection shard at a time.

A distinctive and sometimes surprising piece is **code generation**: Marten
generates C# for document storage and projection handling. By default this
happens at runtime on first use ("dynamic"), which adds cold-start latency; for
production it can be pre-generated ahead of time (`Optimized`/`Static` modes with
`dotnet run -- codegen write`) to remove the warm-up cost[^6].

## Production Notes

**Schema management is a footgun.** `AutoCreate.CreateOrUpdate` (and `All`) let
Marten alter your database at startup. This is convenient in development and
dangerous in production; the recommended path is `AutoCreate.None` plus explicit
migrations generated via `dotnet run -- marten-apply` or a checked-in SQL
script[^6]. Duplicated/computed indexes and new document types can trigger table
rewrites on large tables.

**Serializer choice is effectively permanent.** Switching between Newtonsoft and
System.Text.Json, or changing STJ casing/enum settings, changes how already-stored
JSONB deserializes. Decide early; migrating live data later means rewriting rows.

**Async projections need operational care.** The daemon must run somewhere, needs
its own progress tables, and rebuilds ("projection rebuild") re-read the entire
event store — expensive on large streams and typically an offline or
carefully-throttled operation. Leader election via advisory locks means multiple
app instances are fine, but you must actually start the daemon (`AddAsyncDaemon`)
or async projections silently never advance.

**PLV8 is on the way out.** Older patching relied on the PLV8 Postgres extension
(JavaScript stored procedures). Since v7 Marten ships **native patching** in SQL,
and PLV8-based patching is deprecated; new code should use the native patching
API and avoid the PLV8 install entirely[^1].

**Postgres is the scaling ceiling.** Reads scale with replicas, but writes are
single-primary. Very high-throughput event append or document write workloads hit
Postgres limits, not Marten limits. Multi-tenancy helps isolation (conjoined
single-schema with a tenant column, or database-per-tenant) but does not remove
the single-writer constraint per database.

## When to Use / When Not

**Use when:**
- You are a .NET shop already running PostgreSQL and want documents and/or event
  sourcing without adopting a second datastore.
- You need documents and events changed in one ACID transaction.
- You want event sourcing with projections but prefer Postgres over a dedicated
  event-store product.

**Avoid when:**
- You need a horizontally sharded / distributed document store — reach for
  MongoDB or Cosmos DB.
- Your data is fundamentally relational with heavy joins and reporting — EF Core
  or plain SQL fits better than JSONB documents.
- You are not on .NET, or you cannot run PostgreSQL.

## Alternatives

- dotnet/efcore — use instead when your model is relational and you want a
  full ORM with migrations rather than JSONB documents.
- mongodb/mongo-csharp-driver — use instead when you want a purpose-built,
  horizontally scalable document database and don't need Postgres or a bundled
  event store.
- kurrent-io/EventStore — use instead when event sourcing is the whole product
  and you want a dedicated event-store engine over a Postgres library.
- ravendb/ravendb — use instead when you want a self-contained .NET document
  database with its own storage engine and indexing.
- JasperFx/wolverine — not a replacement but the Critter Stack companion; pair
  with Marten for messaging, sagas, and the transactional outbox.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-09 | First stable release: document DB + early event store on Postgres[^1]. |
| 2.0 | 2017 | Serializer and LINQ improvements. |
| 3.0 | 2018 | Async and schema-management maturation. |
| 4.0 | 2021-09 | Major rewrite of event sourcing and projections; new code-gen model[^5]. |
| 5.0 | 2022 | Npgsql 6, .NET 6 alignment. |
| 6.0 | 2023 | Npgsql 7/8, .NET 8 support. |
| 7.0 | 2024 | Native SQL patching; PLV8 patching deprecated[^1]. |
| 8.0 | 2025 | Continued Critter Stack alignment and API cleanup. |

## References

[^1]: Marten README and documentation. https://martendb.io/
[^2]: The Critter Stack (Marten + Wolverine). https://wolverinefx.net/ and https://jasperfx.net/
[^3]: JasperFx Software support plans. https://jasperfx.net/support-plans/
[^4]: Marten serialization docs. https://martendb.io/documents/json.html
[^5]: Marten projections / event store docs. https://martendb.io/events/projections/
[^6]: Marten code generation and schema management. https://martendb.io/configuration/prebuilding.html

## Tags

dotnet, csharp, postgresql, document-database, event-sourcing, jsonb, cqrs, orm-alternative, critter-stack, persistence
