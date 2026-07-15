# riok/mapperly

> A Roslyn source generator that writes .NET object-to-object mapping code at compile time, with no runtime reflection.

[GitHub repo](https://github.com/riok/mapperly) ·
[Official website](https://mapperly.riok.app) ·
[License: Apache-2.0](https://github.com/riok/mapperly/blob/main/LICENSE)

## Overview

Mapperly is a C# source generator, maintained by the Swiss consultancy riok[^1], that generates DTO/entity mapping code during compilation. You declare a `partial` class annotated with `[Mapper]` and empty `partial` mapping methods; Mapperly's Roslyn generator fills in the method bodies by matching properties by name and type. The output is ordinary C# — visible in your IDE, steppable in a debugger, and free of any runtime dependency beyond a small abstractions assembly[^2].

The project exists as a reaction to the dominant runtime mappers, above all AutoMapper. Those libraries build mapping plans at startup using reflection and compiled expression trees. That approach is flexible but has three recurring costs: a startup/first-call warmup, opacity (mapping failures surface as runtime exceptions, often deep in a call stack), and hostility to Native AOT and IL trimming, where reflection-heavy code is unsafe or stripped. Mapperly moves all of that to build time. A missing member, an ambiguous mapping, or an unmappable type becomes a compiler diagnostic (the `RMG` code family) rather than a production surprise[^3].

The defining tension is expressiveness versus compile-time guarantees. Mapping is configured through attributes and method signatures, so anything that a runtime mapper expresses as arbitrary C# — conditional logic, data-driven mapping decisions, runtime-registered profiles — is either awkward or must be pushed into user-defined methods. In exchange you get zero reflection, AOT compatibility, and mappings you can read. With ~4,100 stars, 200+ forks, and commits landing the same week as this writing, it is an actively maintained project rather than an experiment.

## Getting Started

```bash
dotnet add package Riok.Mapperly
```

```csharp
using Riok.Mapperly.Abstractions;

public class Car    { public int NumberOfSeats { get; set; } public string Make { get; set; } }
public class CarDto { public int NumberOfSeats { get; set; } public string Make { get; set; } }

[Mapper]
public partial class CarMapper
{
    public partial CarDto CarToCarDto(Car car);   // body is generated at build time
}

// usage
var dto = new CarMapper().CarToCarDto(new Car { NumberOfSeats = 5, Make = "Volvo" });
```

The generated `CarToCarDto` is a plain method that news up a `CarDto` and assigns each property. Right-click "Go to Definition" on the partial method to read it.

## Architecture / How It Works

Mapperly ships as an analyzer/source-generator package (`Riok.Mapperly`) plus an abstractions assembly holding the attributes. There is no runtime library to reference in production output. The generator runs inside the C# compiler:

1. It finds every `[Mapper]` class and enumerates the `partial` methods declared on it.
2. For each method it resolves the source and target types and builds a mapping plan by matching members by name (case-insensitive by default), recursing into nested objects, collections, and dictionaries, and generating dedicated methods for each pairing.
3. It emits C# for every method body and reports diagnostics for anything it cannot resolve.

Configuration is entirely attribute-driven. `[MapProperty]` redirects or renames a member (including flattening across `.` paths); `[MapperIgnoreTarget]` / `[MapperIgnoreSource]` suppress the "unmapped member" diagnostic; `[MapValue]` injects constants; `[MapDerivedType]` handles inheritance/polymorphic hierarchies; enum mapping strategy (by value, by name, explicit) is set on `[Mapper]` or per-method. User-defined `private partial` methods, or ordinary methods with matching signatures, are picked up and called for types Mapperly cannot map itself — the escape hatch for custom logic.

Because it is a generator, the incremental-generation model matters. Mapperly is written as an incremental source generator, so edits re-run only the affected mappers rather than the whole compilation. Diagnostics and generated output appear as you type in a Roslyn-aware IDE, but the picture can go stale after large refactors or package upgrades, where a clean rebuild is the reliable reset.

## Production Notes

- **Errors are diagnostics, not exceptions.** Unmapped target members are warnings by default, not build failures. Teams that want mapping gaps to fail the build must promote the relevant `RMG` diagnostics to errors in `.editorconfig` / `TreatWarningsAsErrors`. Left at defaults, a newly added DTO field can silently stay `null`/default.
- **Reference loops need opt-in.** Self-referential or cyclic graphs will produce stack overflows unless reference handling is enabled (`[Mapper(UseReferenceHandling = true)]`), which adds bookkeeping overhead. It is off by default for performance.
- **Enum mapping is a quiet footgun.** The default strategy maps enums by underlying value. Two enums that "look" alike but have divergent numeric backing map to wrong members with no error. Choosing by-name explicitly is safer for loosely coupled enums.
- **AOT / trimming is the headline win.** The generated code contains no reflection, so it survives Native AOT and aggressive IL trimming intact — the main reason projects migrate off AutoMapper, whose runtime model is trim-unsafe.
- **IDE staleness.** Generated members can lag behind rapid edits or immediately after a package bump; symptoms are phantom "method has no body" errors that vanish on rebuild. Budget for occasional clean builds in CI and locally.
- **Two release channels.** A stable channel under semantic versioning and a `next` preview channel that explicitly does not honor semver[^2]. Do not pin production code to `next` builds expecting stability.
- **Support policy is narrow.** Only the latest stable release is fully supported, tracking currently-supported .NET versions[^2]. Long-lived LTS apps that cannot chase the newest Mapperly may run unsupported.

## When to Use / When Not

**Use when:**
- You target Native AOT, trimming, or Blazor WASM, where reflection-based mappers are unsafe.
- You want mapping mistakes caught at build time and mapping code you can actually read.
- You want zero runtime mapping dependency and no startup warmup cost.
- Your mappings are largely structural (rename, flatten, nest, collections, enums).

**Avoid when:**
- Your mapping decisions are genuinely dynamic or configured at runtime.
- You need heavy conditional/business logic inside mappings — hand-written code or a runtime mapper may read better than a wall of attributes and partial helpers.
- You are on an old toolchain: source generators need a recent Roslyn/.NET SDK, and Mapperly supports only its latest release.

## Alternatives

- AutoMapper/AutoMapper — the long-time incumbent, reflection- and expression-based; reach for it when mappings are configured dynamically or you have a large existing profile investment. Note it moved toward commercial licensing, a common trigger for evaluating Mapperly[^4].
- MapsterMapper/Mapster — fast runtime mapping with an optional source-generator tool; a middle ground when you want runtime flexibility plus codegen for hot paths.
- agileobjects/AgileMapper — runtime mapper focused on zero-configuration deep clones and complex object graphs; use when convention-over-configuration matters more than AOT.
- Hand-written mapping (no library) — for small or highly custom surfaces, explicit assignment code has no diagnostics, no generator, and no version churn; Mapperly mainly pays off once the mapping count grows.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2022-02 | Initial development by riok as a Roslyn source-generator mapper[^1]. |
| 1.x | 2023 | First stable line; core attribute API (`[Mapper]`, `[MapProperty]`, flattening, enums, collections). |
| 2.x | 2024 | Later major line; expanded configuration, derived-type/polymorphic mapping, reference handling. |
| 3.x | 2025 onward | Continued majors under semver with per-release upgrade guides in the docs[^5]. |

## References

[^1]: riok — maintaining organization. https://riok.ch
[^2]: Mapperly README, "Release Channels" and "Support Policy". https://github.com/riok/mapperly
[^3]: Mapperly documentation, analyzer diagnostics (`RMG` codes). https://mapperly.riok.app/docs/configuration/analyzer-diagnostics/
[^4]: AutoMapper licensing change discussion (Jimmy Bogard). https://www.jimmybogard.com/automapper-and-mediatr-going-commercial/
[^5]: Mapperly upgrade guides. https://mapperly.riok.app/docs/category/upgrading/

## Tags

csharp, dotnet, source-generator, roslyn, object-mapping, dto, compile-time, native-aot, code-generation, automapper-alternative
