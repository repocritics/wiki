# LuckyPennySoftware/AutoMapper

> Convention-based object-to-object mapper for .NET — the one that eliminates hand-written DTO copying, and the one whose own author later told you to think twice before using.

[GitHub repo](https://github.com/AutoMapper/AutoMapper) ·
[Official website](https://automapper.io) ·
[License: RPL-1.5 OR Commercial](https://github.com/AutoMapper/AutoMapper/blob/main/LICENSE.md)

## Overview

AutoMapper is a .NET library that copies values between object types by convention, created by Jimmy Bogard in 2010[^1]. Its core promise is removing the tedious, mechanical code that maps a domain entity onto a DTO (and back): you declare `CreateMap<Order, OrderDto>()` once, and members with matching names — including flattened paths like `order.Customer.Name` → `CustomerName` — are wired automatically. For a decade it was one of the most-installed packages on NuGet and near-ubiquitous in ASP.NET codebases.

Two things define AutoMapper in 2026. First, a licensing change: in July 2025 the project moved from MIT to a dual Reciprocal Public License 1.5 / commercial license under Lucky Penny Software, and repository ownership moved to the `LuckyPennySoftware` organization[^2][^3]. GitHub still serves the old `AutoMapper/AutoMapper` URL, but the canonical owner is now `LuckyPennySoftware`. Second, a shift in community sentiment — including from Bogard himself, who has publicly cautioned that AutoMapper is often applied to problems it makes worse[^4]. The result is a mature, heavily-embedded library that many new projects now deliberately avoid.

The central tension is that AutoMapper trades explicit, debuggable, compile-checked mapping code for terse configuration and runtime magic. When mappings are simple and stable, that trade pays off. When they aren't, the hidden behavior becomes the most expensive part of the codebase.

## Getting Started

```
dotnet add package AutoMapper
```

```csharp
using AutoMapper;

// 1. Configure at startup (DI-based registration scans assemblies for
//    custom resolvers, converters, and Profile classes)
services.AddAutoMapper(cfg =>
{
    cfg.CreateMap<Order, OrderDto>();
    cfg.CreateMap<Customer, CustomerDto>();
});

// 2. Inject IMapper and execute
public class OrderService(IMapper mapper)
{
    public OrderDto ToDto(Order order) => mapper.Map<OrderDto>(order);
}
```

```csharp
// Validate every configured map has a home for each destination member.
// Wire this into a unit test — it is the only defense against silent gaps.
configuration.AssertConfigurationIsValid();
```

## Architecture / How It Works

At configuration time, every `CreateMap` call is analyzed and compiled into a delegate built from `System.Linq.Expressions` — AutoMapper does not reflect over objects on each `Map` call; it reflects once, builds an execution plan, and compiles it[^5]. This is why the first map of a type pair is comparatively slow and subsequent maps are fast.

Member matching is convention-driven. The default resolver matches destination members to source members by name, then applies **flattening**: a destination `CustomerName` is satisfied by `source.Customer.Name` or a `GetCustomerName()` method. Anything the conventions cannot resolve must be specified explicitly with `.ForMember(...)`, a custom `IValueResolver`, an `ITypeConverter`, or a `Profile` subclass that groups related maps.

`ProjectTo<TDto>()` is a distinct and often under-appreciated feature: instead of materializing entities and mapping them in memory, it rewrites the mapping as an expression tree that composes onto an `IQueryable`, so an EF Core query selects only the destination columns and translates the projection into SQL[^6]. This path has different, stricter rules than in-memory mapping — custom resolvers and any logic that cannot be translated to SQL will not work inside `ProjectTo`.

The configuration is the coupling story. All maps live in a single `MapperConfiguration` graph; the compiled plans hold references across type pairs, so a change to one DTO's shape can quietly alter what a convention resolves elsewhere. The `IMapper` instance is stateless and thread-safe once built, which is why the static `Mapper` facade of early versions was removed in favor of a single injected instance.

## Production Notes

**Licensing is now the first design decision, not an afterthought.** Since v15 (2025-07-02) the code is governed by RPL-1.5 — a strong reciprocal (copyleft) license requiring you to release the source of software built on it — *or* a paid commercial license, activated with a `LicenseKey`[^2][^3]. Versions up to and including the 14.x line remain MIT[^7]. Many teams responded by pinning to 14.x, migrating to a source-generator alternative, or purchasing licenses. Any upgrade past 14.x is a legal decision that must clear procurement, not a routine `dotnet add package`.

**Errors move from compile time to run time.** A forgotten `ForMember`, a renamed source property, or a type mismatch surfaces as a runtime `AutoMapperMappingException`, not a build failure. `AssertConfigurationIsValid()` catches unmapped destination members but must be run in a test — it does nothing if you never call it, and it does not catch *wrong* mappings, only missing ones.

**Debugging cost is real.** Stack traces run through compiled expression trees, so "where did this value come from?" is genuinely hard to answer for non-trivial maps. Flattening is the sharpest footgun: after a refactor, a convention can silently start resolving a *different* member with a matching name, producing wrong data with no error.

**Performance.** Compiled delegates are fast, but not free versus hand-written assignment; for hot paths, source generators like Mapperly (zero runtime reflection) measurably win. `ProjectTo` is the opposite case — it can be dramatically faster than manual mapping because it avoids over-fetching from the database.

**Upgrade friction.** The v9 removal of the static `Mapper` API forced every codebase onto instance-based injection. Major versions periodically tighten configuration validation, so upgrades can turn previously-tolerated ambiguous maps into hard errors.

## When to Use / When Not

**Use when:**
- You have many structurally-similar type pairs (entity ↔ DTO) with stable, name-matched members.
- You use `ProjectTo` with EF Core to push projection into SQL and avoid loading full entities.
- You already have AutoMapper across a large codebase and consistency matters more than removing it.

**Avoid when:**
- Mappings are few or simple — hand-written mapping (or a `record` constructor) is more explicit, debuggable, and compile-checked.
- Mapping logic is complex or conditional — the indirection hides exactly the code you most need to read.
- The RPL-1.5 / commercial terms are unacceptable and you cannot stay on 14.x.
- You want compile-time guarantees; a source generator gives errors at build time instead of run time.

## Alternatives

- riok/Mapperly — source-generator mapper; produces plain C# at compile time, zero runtime reflection, permissive license. Use when you want compile-time safety and no runtime cost.
- MapsterMapper/Mapster — convention mapper with optional code generation and strong runtime performance. Use when you want AutoMapper-style convenience under a permissive license.
- Hand-written mapping — no dependency, fully explicit and debuggable. Use for small or simple projects (the maintainer's own recommendation for many cases[^4]).
- dotnet/efcore — for entity→DTO, `Select` projections in LINQ do the same job as `ProjectTo` without a mapper. Use when the mapping only exists to shape a query result.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010 | Initial release by Jimmy Bogard[^1]. |
| 9.0 | 2019 | Static `Mapper` facade removed; instance `IMapper` only[^5]. |
| 13.0 | 2024-02-05 | Ongoing MIT-era release. |
| 14.0 | 2025-02-14 | Last MIT-licensed major line[^7]. |
| 15.0 | 2025-07-02 | Relicensed to RPL-1.5 / commercial; license keys introduced[^2][^3]. |
| 16.x | 2026-03 → | Current line (16.2.0, 2026-07-02); 15.x still receiving patches. |

## References

[^1]: AutoMapper — repository created 2010-02-06. https://github.com/AutoMapper/AutoMapper
[^2]: LICENSE.md — "Your license to Lucky Penny Software source code and/or binaries is governed by the Reciprocal Public License 1.5 (RPL1.5) ... or ... the License Agreement." https://github.com/AutoMapper/AutoMapper/blob/main/LICENSE.md
[^3]: AutoMapper commercial licensing and license keys. https://automapper.io
[^4]: Jimmy Bogard, "AutoMapper's Design Philosophy" / cautions on overuse. https://www.jimmybogard.com/automappers-design-philosophy/
[^5]: AutoMapper documentation — configuration, expression compilation, and DI. https://docs.automapper.io
[^6]: AutoMapper documentation — Queryable Extensions (`ProjectTo`). https://docs.automapper.io/en/latest/Queryable-Extensions.html
[^7]: Release history — v14.0.0 (2025-02-14) preceded the v15.0.0 relicensing (2025-07-02). https://github.com/AutoMapper/AutoMapper/releases

## Tags

csharp, dotnet, object-mapping, dto, orm-adjacent, convention-based, code-generation-alternative, commercial-license, rpl-1.5, aspnet
