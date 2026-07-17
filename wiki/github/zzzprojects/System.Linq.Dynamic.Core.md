# zzzprojects/System.Linq.Dynamic.Core

> String-based, runtime-built LINQ queries for .NET — the maintained successor to Microsoft's original Dynamic LINQ sample.

[GitHub repo](https://github.com/zzzprojects/System.Linq.Dynamic.Core) ·
[Official website](https://dynamic-linq.net/) ·
[License: Apache-2.0](https://github.com/zzzprojects/System.Linq.Dynamic.Core/blob/master/LICENSE)

## Overview

System.Linq.Dynamic.Core lets you express LINQ operators as strings evaluated at runtime rather than as compile-time lambdas. Instead of `.Where(c => c.City == "London")` you write `.Where("City == @0", "London")`, and the library parses that string into a `System.Linq.Expressions` tree. It is a .NET Standard / .NET Core port of the `System.Linq.Dynamic` sample Microsoft shipped alongside LINQ for .NET 3.5/4.0[^1], extended with unit tests, more operators, and modern packaging.

The project has a tangled lineage: the original Microsoft sample was forked repeatedly (kahanu, NArnott, others), Stef Heyenrath (StefH) built the ".Core" port on top of NArnott's fork, and the repository is now owned and maintained under the ZZZ Projects organization[^2]. The `StefH/...` URL still resolves via redirect to `zzzprojects/System.Linq.Dynamic.Core`, which is why references to both names circulate.

Its defining tension is exactly what makes it useful: queries become data. That enables generic grids, report builders, and API filter endpoints where the shape of a query is unknown until runtime — but it also turns a string into executable expression logic, which is a parsing and security surface the compiler would otherwise close off. Much of the library's recent history is spent narrowing that surface (see Production Notes).

## Getting Started

```bash
dotnet add package System.Linq.Dynamic.Core
```

```csharp
using System.Linq.Dynamic.Core;

// Works on any IQueryable (EF Core) or IEnumerable (LINQ to Objects).
var result = db.Customers
    .Where("City == @0 and Orders.Count >= @1", "London", 10)  // @N = parameters
    .OrderBy("CompanyName")
    .Select("new (CompanyName as Name, Phone)");               // dynamic projection

// Interpolated form (net46+, netcoreapp2.1+, netstandard1.3+):
string city = "London";
db.Customers.WhereInterpolated($"City == {city}");
```

Use the `@0`, `@1` placeholders — never string-concatenate user input into the query text (see Production Notes). `Select("new (...)")` returns `IQueryable` of a dynamically generated type; read columns via `DynamicProperty`/reflection or project into a known DTO.

## Architecture / How It Works

The core is a hand-written recursive-descent parser (`ExpressionParser`) that tokenizes the query string and produces a `System.Linq.Expressions.Expression`. The dynamic operators (`Where`, `OrderBy`, `Select`, `GroupBy`, etc.) are extension methods on `IQueryable`/`IEnumerable` that build a `MethodCallExpression` wrapping that parsed expression and hand it back to the underlying provider.

This is the key architectural fact: the library does **not** execute queries. It only builds expression trees. What happens next depends entirely on the provider:

- On `IQueryable` backed by **EF Core**, the expression tree is passed to the provider's translator, which turns it into SQL. Anything the provider cannot translate fails at execution time or (in older EF versions) silently falls back to client evaluation.
- On `IEnumerable` (LINQ to Objects), the expression is compiled to a delegate via `Expression.Compile()` and run in-process.

Behavior is tuned through a `ParsingConfig` object: it controls the custom type provider, culture, number parsing, EF Core compatibility flags, and the security restrictions described below. Types that dynamic queries are allowed to reference must either be primitives, be annotated with `[DynamicLinqType]`, or be supplied through a custom `IDynamicLinkCustomTypeProvider`[^3]. Dynamic projections (`new (...)`) synthesize a runtime type via `System.Reflection.Emit` on most targets.

## Production Notes

**String queries are an injection surface — treat them like SQL.** The parser evaluates method calls and member access, so an attacker who controls the query string (not just parameter values) can reach further than a plain `WHERE`. Always pass user data through `@N` parameters; never interpolate raw user strings into the predicate text. The project has shipped several security-motivated breaking changes for exactly this reason[^3][^4].

**CVE-2024-51417 and the v1.6.0 clamp-down.** Method calls on the `object` type (including `ToString`/`Equals`) were disabled by default to mitigate a reported vulnerability; re-enable via `AllowEqualsAndToStringMethodsOnObject` only if you understand the exposure[^4]. The same release set `RestrictOrderByToPropertyOrField = true` by default, so `OrderBy("SomeMethod()")` now throws unless you opt out. Teams upgrading across this line frequently hit runtime exceptions on previously-working queries.

**The `[DynamicLinqType]` footgun (v1.3.0+).** Since v1.3.0, calling methods on your own classes from a dynamic string is blocked unless the class carries `[DynamicLinqType]` or is registered through a custom type provider on `ParsingConfig`. This is the single most common "it worked before the upgrade" report.

**EF Core translation is not guaranteed.** Because the library only emits expression trees, whether a dynamic query runs as SQL is the provider's decision. Complex dynamic `Select`/`GroupBy` projections may not translate; test against the actual database, not just LINQ to Objects, and set the EF-compatibility flags on `ParsingConfig` for your EF version.

**Parsing cost.** Every call re-parses the string into an expression tree, and IEnumerable paths also pay `Expression.Compile()`. For hot loops, parse once and reuse, or prefer statically-built expressions. There is no transparent global parse cache to rely on.

**Package sprawl.** The org publishes several NuGet packages against the same engine — `System.Linq.Dynamic.Core`, `EntityFramework.DynamicLinq` (EF6), `Microsoft.EntityFrameworkCore.DynamicLinq`, plus JSON adapters. Pick the one matching your data stack; mixing them or pulling the wrong EF variant causes confusing binding errors.

## When to Use / When Not

**Use when:**
- Query shape is genuinely dynamic at runtime — user-configurable grids, saved filters, generic report/search endpoints.
- You want sort/filter/projection strings driven by an API or UI without a bespoke expression builder.
- You need it to translate to SQL through EF Core rather than pulling rows into memory.

**Avoid when:**
- The query is known at compile time — plain typed LINQ is safer, faster, and refactor-friendly.
- Untrusted end users can control the full predicate string and you cannot strictly whitelist fields/operators.
- You only need filtering/sorting/paging for an HTTP API — a narrower, purpose-built library (below) is a smaller attack surface.

## Alternatives

- alirezanet/Gridify — string filter/sort/paging for web APIs with a compact query syntax; narrower surface than full dynamic LINQ.
- Biarity/Sieve — ASP.NET Core sorting/filtering/pagination driven by query-string parameters; opt-in field whitelisting by design.
- dynamicexpresso/DynamicExpresso — interpreter for C# expressions from strings; use when you need general expression evaluation, not IQueryable operators.
- scottksmith95/LINQKit — `PredicateBuilder` and expression stitching for dynamic `Where` while staying strongly typed.
- Hand-built System.Linq.Expressions — verbose but keeps everything compile-checked and closes the string-parsing hole entirely; use when queries come from a fixed set of code paths.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-04-08 | Repo created as the .NET Core / Standard port, forked from the NArnott / Microsoft Dynamic LINQ lineage[^1][^2]. |
| 1.3.0 | ~2020 | Security change: methods only callable on predefined types; own classes require `[DynamicLinqType]`[^3]. |
| 1.6.0 | ~2024 | CVE-2024-51417 mitigation: `object` methods (incl. `ToString`/`Equals`) blocked by default; `RestrictOrderByToPropertyOrField` defaults to `true`[^4]. |
| — | 2026-07 | Actively maintained under ZZZ Projects; broad TFM coverage from net35 through net9.0. |

## References

[^1]: Scott Guthrie, "Dynamic LINQ" — the original Microsoft sample this project derives from. http://weblogs.asp.net/scottgu/dynamic-linq-part-1-using-the-linq-dynamic-query-library
[^2]: Repository README, "Fork details" — StefH fork lineage and ZZZ Projects stewardship. https://github.com/zzzprojects/System.Linq.Dynamic.Core
[^3]: Dynamic LINQ docs, `DynamicLinqType` attribute and `CustomTypeProvider`. https://dynamic-linq.net/advanced-extending#dynamiclinqtype-attribute
[^4]: Repository README, "Breaking changes" (v1.3.0 / v1.6.0), referencing CVE-2024-51417. https://github.com/zzzprojects/System.Linq.Dynamic.Core#breaking-changes

## Tags

csharp, dotnet, linq, dynamic-linq, iqueryable, entity-framework-core, expression-trees, query-builder, runtime-queries, apache-2.0
