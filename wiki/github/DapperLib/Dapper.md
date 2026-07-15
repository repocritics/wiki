# DapperLib/Dapper

> A micro-ORM for .NET: extension methods on ADO.NET connections that map SQL results to objects, with no query language, change tracking, or schema of its own.

[GitHub repo](https://github.com/DapperLib/Dapper) ·
[Official website](https://www.learndapper.com/) ·
[License: Apache-2.0](https://github.com/DapperLib/Dapper/blob/main/License.txt)

## Overview

Dapper is a "micro-ORM" — a thin mapping layer over raw ADO.NET rather than a full object-relational mapper. It was written at Stack Overflow (originally by Sam Saffron and Marc Gravell) around 2011 to replace a slower LINQ-to-SQL data path on one of the busiest sites on the web[^1]. It adds extension methods such as `Query<T>`, `QuerySingle<T>`, and `Execute` to any `IDbConnection`/`DbConnection`, runs the SQL you hand it, and materializes rows into your types. It does not generate SQL, track changes, manage migrations, or model relationships. You write the SQL; Dapper handles parameterization and object mapping.

The defining tradeoff is **control versus convenience**. Because Dapper is a direct pass-through to ADO.NET, it is close to hand-written `SqlCommand` in throughput and allocation, and it works with any ADO.NET provider (SQL Server, PostgreSQL via Npgsql, MySQL, SQLite, Oracle). What you give up is everything a full ORM like Entity Framework Core does for you: there is no `IQueryable` LINQ provider, no unit of work, no identity map, no lazy loading, no schema tracking. Every query is a string, and every mistake in that string is a runtime error, not a compile error.

Dapper sits at the "you already know SQL and want it to stay visible" end of the .NET data-access spectrum. It is heavily used in high-throughput services, read-mostly query paths, reporting, and codebases that treat SQL as a first-class artifact. It is frequently paired with a full ORM in the same project — EF Core for writes and aggregates, Dapper for hot read paths.

## Getting Started

```bash
dotnet add package Dapper
# provider of your choice, e.g.:
dotnet add package Microsoft.Data.SqlClient   # or Npgsql, MySqlConnector, etc.
```

```csharp
using System.Data;
using Dapper;
using Microsoft.Data.SqlClient;

public record Dog(int Id, string Name, int? Age, float? Weight);

using IDbConnection db = new SqlConnection(connectionString);

// Parameters come from an anonymous object; @Age/@Id are real DB parameters.
var dogs = db.Query<Dog>(
    "select Id, Name, Age, Weight from Dogs where Age >= @MinAge",
    new { MinAge = 3 });

// Single-row helpers: QuerySingle / QueryFirst (+ OrDefault variants)
var one = db.QuerySingleOrDefault<Dog>(
    "select * from Dogs where Id = @Id", new { Id = 42 });

// Non-query; returns rows affected. Pass an array to execute N times.
int affected = db.Execute(
    "insert into Dogs (Name, Age) values (@Name, @Age)",
    new[] { new { Name = "Rex", Age = 4 }, new { Name = "Fido", Age = 2 } });
```

Async variants (`QueryAsync`, `ExecuteAsync`, etc.) mirror the synchronous API.

## Architecture / How It Works

Dapper is a single assembly of extension methods; there is no runtime engine to configure. The interesting machinery is in two places: **parameter expansion** and **row materialization**.

**Row materialization via emitted IL.** For each distinct query shape, Dapper builds a deserializer that reads an `IDataReader` row and constructs your object. It generates this deserializer once using `System.Reflection.Emit` (`DynamicMethod`), then caches and reuses the compiled delegate[^2]. The cache key combines the SQL text, the command type, the target type, the parameter object's type, and the connection string. This emit-and-cache design is why Dapper's mapping cost approaches hand-written ADO.NET after warm-up: the per-row path is compiled IL, not reflection.

**Parameter handling.** Parameters are supplied as anonymous objects, `Dictionary<string, object>`, or a `DynamicParameters` instance. Dapper reads the object's properties and adds real ADO.NET parameters (`@Name`, `@Age`), so values are never string-concatenated into SQL. It also rewrites `IN` clauses: `where Id in @Ids` with `new { Ids = new[]{1,2,3} }` is expanded to `in (@Ids1, @Ids2, @Ids3)` with three bound parameters[^3].

**Multi-mapping and multiple result sets.** `Query<TFirst, TSecond, TReturn>(..., splitOn: "...")` maps a single wide row into several objects, splitting columns at the named boundary (default `"Id"`). `QueryMultiple` returns a `GridReader` for reading several result sets from one command in sequence.

**Buffered vs unbuffered.** By default `Query<T>` is *buffered*: the entire result set is read into a `List<T>` before returning, and the reader/connection is released. Passing `buffered: false` streams rows lazily via `IEnumerable<T>`, which lowers peak memory but holds the reader (and connection) open until you finish enumerating. Newer versions add `QueryUnbufferedAsync` returning `IAsyncEnumerable<T>` for async streaming.

Companion packages live in the same repo: `Dapper.SqlBuilder` (composable dynamic SQL), `Dapper.Rainbow` (thin CRUD helper), `Dapper.EntityFramework` (EF type handlers), and `*.StrongName` variants for signed-assembly consumers. `Dapper.Contrib` (basic attribute-driven CRUD) is historically part of the family. Note that **Dapper Plus** — a commercial bulk-operations product and a project sponsor — is a separate paid library, not this repo.

## Production Notes

**SQL is strings; injection is your responsibility.** Dapper parameterizes anything you pass as an argument, which is safe. The danger is *literal replacements* (`{=Admin}`) and any manual string interpolation of SQL. Literal replacement is limited to `bool`/numeric types and is inlined into the SQL text, not sent as a parameter — never route user input through it or through `$"..."` interpolation.

**The query cache can grow unbounded with dynamic SQL.** Because the cache is keyed on SQL text, code that builds unique SQL strings per call (e.g., concatenating variable `IN` lists as literals, or interpolating filters) produces a new cache entry every time and can leak memory. Prefer stable SQL with parameters. Dapper exposes `SqlMapper.GetCachedSQLCount()` and `SqlMapper.PurgeQueryCache()` for diagnosis and manual eviction.

**Column-to-property mapping is exact by default.** Matching is case-insensitive but does not translate naming conventions: a `first_name` column will not map to a `FirstName` property unless you opt in with `DefaultTypeMap.MatchNamesWithUnderscores = true`, alias in SQL (`select first_name as FirstName`), or register a custom `ITypeMap`. Silent nulls from unmatched columns are a common "why is this property empty" bug.

**Multi-mapping `splitOn` is a frequent footgun.** The split defaults to `Id`; if your joined result has a different boundary column (or multiple `Id` columns in the wrong order), rows map incorrectly with no error. Order columns deliberately and set `splitOn` explicitly.

**No compile-time safety.** Typos in column names, arity mismatches, and schema drift surface only at runtime. Teams usually compensate with integration tests that hit a real database and with SQL kept in reviewable, greppable locations.

**NativeAOT and trimming.** Classic Dapper relies on `Reflection.Emit` and runtime reflection, which are incompatible with IL trimming and NativeAOT. For AOT/trimmed deployments the maintainers ship a separate source-generator project, **Dapper.AOT** (a distinct repo), which generates the mapping code at build time instead of emitting it at runtime[^4].

**Bulk inserts are loops, not bulk.** `Execute` with an array runs the command once per element (still a round-trip each unless batched by the provider). For large loads use provider bulk-copy (`SqlBulkCopy`), table-valued parameters, or a dedicated bulk library rather than Dapper's multi-execute.

## When to Use / When Not

**Use when:**
- You know SQL and want it to stay visible, reviewable, and tunable.
- You have hot read paths where ORM overhead (tracking, translation) is measurable.
- You target multiple/awkward databases and want direct ADO.NET provider control.
- You want a near-zero-config dependency with a tiny surface area and near-hand-coded performance.

**Avoid when:**
- You want writes, aggregates, and relationships managed for you (change tracking, unit of work) — reach for a full ORM.
- You want compile-time-checked queries and LINQ composition over the schema.
- Your team doesn't want to write and maintain SQL by hand.
- You need NativeAOT/trimmed output without adopting the separate Dapper.AOT generator.

## Alternatives

- dotnet/efcore — full ORM with LINQ provider, change tracking, and migrations; use when you want the database abstracted rather than exposed.
- linq2db/linq2db — a LINQ-to-SQL provider that stays close to SQL semantics; use when you want composable typed queries without EF's tracking machinery.
- mikependon/RepoDB — hybrid micro-ORM with built-in CRUD, caching, and bulk operations; use when you want Dapper-like speed plus batteries.
- FransBouma/RawDataAccessBencher — not an ORM but the reference benchmark suite; use to compare data-access options on your own hardware.
- DapperLib/DapperAOT — the same programming model via source generators; use when you need trimming/NativeAOT compatibility.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011 | Created at Stack Overflow to speed up a hot data path; repo first pushed 2011-04-14[^1]. |
| 1.x | 2011–2019 | Long-lived 1.x line as `StackExchange/Dapper`; core `Query`/`Execute` API stabilized. |
| 2.0 | 2020 | Package restructuring; `Dapper.StrongName` split for signed-assembly consumers. |
| 2.1 | 2023– | Current line (latest 2.1.79); added async unbuffered streaming (`QueryUnbufferedAsync`). |
| — | ~2021 | Repository moved from `StackExchange/Dapper` to the `DapperLib` organization. |

## References

[^1]: Sam Saffron, "How I learned to stop worrying and write my own ORM" — background on Dapper's origin at Stack Overflow. https://samsaffron.com/archive/2011/03/30/How+I+learned+to+stop+worrying+and+write+my+own+ORM
[^2]: Dapper source, `SqlMapper` deserializer emit / query cache. https://github.com/DapperLib/Dapper/blob/main/Dapper/SqlMapper.cs
[^3]: Dapper README, list/`IN` expansion and parameterization. https://github.com/DapperLib/Dapper/blob/main/Readme.md
[^4]: Dapper.AOT — build-time source-generated Dapper for trimming/NativeAOT. https://github.com/DapperLib/DapperAOT
[^5]: Official documentation and tutorials. https://www.learndapper.com/

## Tags

csharp, dotnet, micro-orm, orm, ado-net, sql, database, data-access, object-mapper, performance
