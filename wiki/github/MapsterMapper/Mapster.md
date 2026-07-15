# MapsterMapper/Mapster

> A convention-based object-to-object mapper for .NET that compiles mappings to expression-tree delegates at runtime — the main lightweight alternative to AutoMapper.

[GitHub repo](https://github.com/MapsterMapper/Mapster) ·
[Official website](http://mapstermapper.github.io/Mapster/) ·
[License: MIT](https://github.com/MapsterMapper/Mapster/blob/master/LICENSE)

## Overview

Mapster maps values between .NET objects — typically entity ↔ DTO, or DTO ↔ view model — so you don't hand-write assignment code. The core is a single extension method, `Adapt<T>()`, backed by mapping logic that Mapster builds as a LINQ expression tree and compiles to a delegate on first use. By default it matches members by name and type using conventions, so the common case needs no configuration at all[^1].

The project has a two-part history. It was created in 2014[^2] by Eric Swann as a small reflection-based mapper; the current engine is a later expression-tree rewrite maintained by Chaowlert Chaisrichalermpol under the MapsterMapper organization. Around that rewrite Mapster positioned itself explicitly against AutoMapper — same convention-driven ergonomics, lower per-map overhead, and no mandatory central configuration. Its published `FlatTypes` benchmark shows a simple `Person -> PersonDTO` map running several times faster than AutoMapper 14 at equal allocations, and roughly on par with the source-generator mapper Mapperly[^3].

The defining tension is convention versus verification. Name-based mapping that "just works" is also mapping that silently does the wrong thing when a property is renamed, a type subtly differs, or a new field is added and quietly left unmapped. Mapster's answer is a spectrum — pure runtime convention at one end, explicit fluent config in the middle, and compile-time code generation (`Mapster.Tool`) at the other end, where mappings become inspectable, debuggable C# validated by the compiler.

## Getting Started

```bash
dotnet add package Mapster
```

```csharp
using Mapster;

// Map to a new instance (convention-based, no config)
var dto = student.Adapt<StudentDto>();

// Map onto an existing instance
student.Adapt(existingDto);

// Custom rules via global config
TypeAdapterConfig<Student, StudentDto>
    .NewConfig()
    .Map(dest => dest.FullName, src => $"{src.First} {src.Last}")
    .Ignore(dest => dest.Internal);

// IQueryable projection — translated to SQL by the LINQ provider
var rows = db.Students
    .ProjectToType<StudentDto>()   // becomes a Select() in the query
    .ToList();
```

For DI, `Mapster.DependencyInjection` registers an `IMapper` so call sites depend on an interface rather than the static `Adapt` extension:

```csharp
services.AddMapster();               // then inject IMapper
```

## Architecture / How It Works

Mapster's runtime path builds, per source→destination pair, an `Expression<Func<TSource, TDestination>>` that assigns each matched member, then compiles it to a delegate and caches it. The first map of a given type pair pays a compile cost; subsequent maps run the cached delegate with no reflection. This is the same broad strategy as modern AutoMapper, and the reason both are far faster than naive reflection-based copying.

Configuration lives in `TypeAdapterConfig`. There is a process-wide `TypeAdapterConfig.GlobalSettings` singleton that `Adapt` uses by default, plus the option to construct isolated `TypeAdapterConfig` instances and pass them explicitly. Rules are attached fluently (`Map`, `Ignore`, `IgnoreNullValues`, `PreserveReference` for cycles, `MaxDepth`, `AfterMapping`, etc.) or by attributes.

`ProjectToType<T>()` is a distinct code path. Instead of compiling a mapping delegate, it produces the projection expression and hands it to the `IQueryable` provider (typically EF Core), which translates it into the SQL `SELECT`. This means projection never materializes the full entity — but it is also bound by what the query provider can translate, which is a strict subset of what the in-memory mapper supports.

The compilation backend is pluggable. The default uses `Expression.Compile`; optional packages swap in FastExpressionCompiler (FEC) or a Roslyn-based compiler for different first-use/throughput tradeoffs. The other axis is `Mapster.Tool`, a dotnet CLI tool that reads `[AdaptTo]` / `[GenerateMapper]` attributes and emits mapper classes and DTO models as C# source at build time, moving the work entirely out of runtime[^4].

Supporting packages orbit the core: `Mapster.EFCore` / `Mapster.EF6` for ORM integration, `Mapster.Immutable` for immutable collections, `Mapster.JsonNet`, and `Mapster.Diagnostics` / `ExpressionDebugger` for step-into debugging of generated mapping code.

## Production Notes

- **Silent partial mappings are the primary footgun.** Convention mapping does not error when a destination member has no source match — it leaves it at its default. Adding a DTO field and forgetting the mapping produces a null/zero, not an exception. Use `TypeAdapterConfig.GlobalSettings.Compile()` at startup, or explicit config with `.Compile()`, to surface mapping problems early; code generation makes them compile errors.
- **First-use compile latency.** Because delegates compile on first map of each type pair, cold requests pay a one-time cost. For latency-sensitive services, precompile at startup (`GlobalSettings.Compile()`), or use `Mapster.Tool` codegen to eliminate runtime compilation entirely.
- **The global static config hurts test isolation.** `GlobalSettings` is process-wide mutable state. Tests that register configs can bleed into each other; parallel test runs are the usual place this surfaces. Prefer scoped `TypeAdapterConfig` instances, or reset/`Clear` between tests.
- **`ProjectToType` ≠ `Adapt`.** A mapping config that works in memory can throw or silently fall back to client evaluation under a LINQ provider. Custom `Map` expressions, method calls, and conditional logic frequently don't translate to SQL. Keep projection configs simple and test the generated query.
- **Two-way and nested inference can surprise.** Flattening/unflattening conventions and reverse mappings sometimes match members you didn't intend. For anything non-trivial, make the mapping explicit rather than trusting inference.
- **Debuggability.** Runtime expression mapping is opaque when it misbehaves; you cannot step into it by default. `Mapster.Diagnostics`/`ExpressionDebugger` or switching to generated code is effectively required for diagnosing wrong-value bugs in complex maps.
- **Docs lag features.** The engine has more capability than the site documents cleanly; answers often live in GitHub issues and the wiki rather than the reference docs.

## When to Use / When Not

**Use when:**
- You want AutoMapper-style convention mapping with lower overhead and no forced central config.
- You need `IQueryable` projection that pushes shaping into the database via EF.
- You want the option to graduate from runtime mapping to compile-time generated mappers without changing the call-site API.

**Avoid when:**
- You want mapping errors caught purely at compile time with zero runtime reflection — a dedicated source generator (Mapperly) is a better fit than Mapster's runtime default.
- Your mappings are few and simple — hand-written mapping is more explicit, more debuggable, and has no dependency or startup cost.
- You need heavy custom logic inside `ProjectToType` — provider translation limits will bite; write the projection by hand.

## Alternatives

- automapper/AutoMapper — the incumbent; larger ecosystem and docs, but heavier and its licensing/commercial direction shifted in 2025 — use it when team familiarity and third-party integration outweigh runtime cost.
- riok/mapperly — Roslyn source generator, all mapping resolved at compile time with no runtime reflection; use it when you want compile-time safety and zero startup cost over runtime convenience.
- Manual mapping (no library) — use when there are few types and you value explicit, greppable, dependency-free code.
- Tim-Maes/Facet — newer projection/DTO-generation library benchmarked alongside Mapster; use it when its compiled-projection model fits your shape.
- dotnet/runtime records + `with` expressions — use for shallow copies of records where a full mapper is overkill.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-05-30 | Repository created; original reflection-based mapper by Eric Swann[^2]. |
| 3.x | ~2019 | Expression-tree engine rewrite under MapsterMapper/Chaowlert; modern `Adapt` core. |
| 7.x | ~2021 | `Mapster.Tool` source generation and fluent codegen API introduced[^4]. |
| 10.x | 2026 | Current line; added compiler variants (Roslyn/FEC) and expanded collection support (`ISet`, `IDictionary`)[^1]. |

## References

[^1]: Mapster README and documentation — MapsterMapper/Mapster. https://github.com/MapsterMapper/Mapster
[^2]: GitHub repository metadata, `created_at` 2014-05-30; MapsterMapper organization. https://github.com/MapsterMapper/Mapster
[^3]: Mapster benchmark results (`FlatTypes` scenario vs AutoMapper 14, Mapperly, Facet). https://mapstermapper.github.io/Mapster/articles/benchmarks.html
[^4]: Mapster.Tool overview — code generation. https://mapstermapper.github.io/Mapster/articles/tools/mapster-tool/Mapster-Tool-Overview.html

## Tags

csharp, dotnet, object-mapping, dto, code-generation, expression-trees, automapper-alternative, ef-core, projection, source-generator
