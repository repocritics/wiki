# khellang/Scrutor

> Assembly scanning and decorator registration for Microsoft.Extensions.DependencyInjection — the two features the built-in container deliberately left out.

[GitHub repo](https://github.com/khellang/Scrutor) ·
[NuGet](https://www.nuget.org/packages/Scrutor) ·
[License: MIT](https://github.com/khellang/Scrutor/blob/master/LICENSE)

## Overview

Scrutor is a small C# library by Kristian Hellang that adds two capabilities to `IServiceCollection`, the registration surface of `Microsoft.Extensions.DependencyInjection` (MS.DI): convention-based assembly scanning (`Scan`) and the decorator pattern (`Decorate`)[^1]. It is not a container. It emits ordinary `ServiceDescriptor` entries into the same collection the built-in provider consumes, so it composes with everything already registered and disappears at runtime.

The reason Scrutor exists is that MS.DI was designed as a minimal, deliberately un-opinionated container. It has no auto-registration ("register every `IHandler` in this assembly") and no decoration ("wrap every `IRepository` in a caching layer"), because Microsoft's stance is that the abstraction is a lowest common denominator and richer behavior belongs in third-party containers like Autofac or Lamar[^2]. Scrutor's bet is the opposite: most teams want those two specific features but do not want to swap out the whole container to get them. As of 2026 the repo sits around 4,300 stars with steady maintenance and low commit velocity — a mature, feature-complete utility rather than an actively expanding framework.

The defining tension is scope. Scrutor is powerful enough to hide a lot of registration logic behind fluent scanning chains, which is convenient at write time and opaque at debug time. A misfiltered `AddClasses` predicate produces no error — it simply registers nothing, or the wrong lifetimes — and the failure surfaces later as a resolution exception far from the cause.

## Getting Started

```bash
dotnet add package Scrutor
```

```csharp
using Scrutor;

var services = new ServiceCollection();

// Convention scanning: register every non-abstract class implementing
// an interface, as that interface, with a chosen lifetime.
services.Scan(scan => scan
    .FromAssemblyOf<IGreeter>()
        .AddClasses(c => c.AssignableTo<IGreeter>())
            .AsImplementedInterfaces()
            .WithScopedLifetime());

// Decoration: wrap an already-registered service.
services.AddSingleton<IGreeter, ConsoleGreeter>();
services.Decorate<IGreeter, LoggingGreeter>();
// Resolving IGreeter now yields LoggingGreeter -> ConsoleGreeter.

var provider = services.BuildServiceProvider();
```

Open generics are supported in both APIs — `AssignableTo(typeof(IHandler<>))` for scanning, and `Decorate(typeof(IHandler<>), typeof(LoggingHandler<>))` for wrapping every closed variant.

## Architecture / How It Works

**Scanning** is a builder that walks assemblies via reflection, applies the type filters you chain (`AssignableTo`, `InNamespaces`, `WithAttribute`, `Where`), and for each surviving type emits `ServiceDescriptor`s according to the "register as" selector (`AsImplementedInterfaces`, `AsSelf`, `As<T>`, `AsMatchingInterface`) and lifetime. It runs once, at startup, when you call `Scan`. There is no runtime component. By default compiler-generated types are excluded from `AddClasses`; UI frameworks that emit generated view classes (Avalonia, WPF) require opting in via `WithAttribute<CompilerGeneratedAttribute>()`[^1].

**Decoration** is the more subtle half. `Decorate<TService, TDecorator>()` searches the existing collection for descriptors registered against `TService`, and replaces each with a new descriptor whose implementation factory (a) resolves the *original* implementation, then (b) constructs the decorator via `ActivatorUtilities`, injecting the original instance plus any other constructor dependencies from the provider. Because it mutates existing descriptors, decoration is order-sensitive: the target service must already be registered when `Decorate` runs, or it throws (`DecorationException` / a missing-registration error). `TryDecorate` is the non-throwing variant for optional decoration.

The decorator's lifetime is inherited from the descriptor it replaces — you do not, and cannot, specify a new lifetime on the decorator. This matters: a decorator over a singleton is effectively a singleton, and captive-dependency rules still apply to whatever it pulls from the provider.

Because everything is expressed as standard `ServiceDescriptor`s, Scrutor is container-agnostic in principle — anything that reads `IServiceCollection` (including third-party containers configured to consume it) sees the results. In practice it is validated against MS.DI's own provider.

## Production Notes

- **Silent no-ops are the main footgun.** A scanning filter that matches zero types registers nothing and reports nothing. Symptoms appear downstream as `Unable to resolve service for type ...`. When a scan "doesn't work," inspect the actual `IServiceCollection` count before and after, or temporarily register `.AsSelf()` to see what was matched.
- **Startup cost.** Scanning is reflection over loaded assemblies. For a handful of assemblies it is negligible; for large solutions that call `FromApplicationDependencies()` (which scans *every* referenced assembly, including the BCL and NuGet packages) it can add measurable startup latency and load assemblies you did not intend to touch. Prefer `FromAssemblyOf<T>` / `FromAssemblies(...)` scoped to your own assemblies.
- **Decoration ordering is load-bearing.** `Decorate` only sees registrations that exist at the moment it runs. If a library registers `IFoo` in its own `AddX()` extension, you must call that before `Decorate<IFoo, ...>()`. Reordering `Program.cs` lines silently changes whether decoration applies.
- **Lifetime surprises.** Decorators adopt the decorated service's lifetime; you cannot promote a transient to a singleton (or vice versa) through decoration. Mixing lifetimes across a decorator chain is a common source of captive-dependency bugs that MS.DI's scope validation will flag only if `ValidateScopes`/`ValidateOnBuild` is enabled.
- **Keyed services.** MS.DI 8.0 introduced keyed services; Scrutor added decoration support for keyed registrations in a later release. If you are on an older Scrutor with a new .NET, keyed decoration may be unavailable — pin versions deliberately.
- **Duplicate/`Try` semantics.** Scanning does not deduplicate against existing registrations the way `TryAdd*` does; scanning an assembly twice, or scanning types also registered by hand, produces multiple descriptors and multi-registration resolution (`IEnumerable<T>`) behavior. Know which selector you used.
- **Reflection and trimming.** Because registration is reflection-driven, Scrutor interacts poorly with aggressive IL trimming / Native AOT: types discovered only via scanning can be trimmed away. AOT-first apps should prefer explicit registration or a source-generator approach.

## When to Use / When Not

**Use when:**
- You are staying on MS.DI (ASP.NET Core default) and want assembly scanning or decoration without adopting a full third-party container.
- You have many same-shaped services (handlers, validators, repositories) that are tedious to register by hand.
- You need a cross-cutting concern (logging, caching, retry) layered over an existing service via the decorator pattern.

**Avoid when:**
- You already use a full container (Autofac, Lamar, Castle Windsor) — they have native scanning and decoration; Scrutor is redundant.
- You target Native AOT or heavy trimming, where reflection-based discovery is fragile.
- Your registration is small and explicit — hand-written `AddScoped` lines are more debuggable than a fluent scan, and reviewers can see exactly what is registered.

## Alternatives

- autofac/Autofac — full-featured IoC container with built-in scanning, decoration, and interception; use instead when you want to replace MS.DI entirely rather than extend it.
- JasperFx/lamar — StructureMap's successor, fast MS.DI-compatible container with rich conventions; use when you want scanning as a first-class container feature.
- castleproject/Windsor — mature container with dynamic-proxy interception; use when you need AOP-style interception beyond simple decoration.
- dotnet/runtime — the `Microsoft.Extensions.DependencyInjection` container itself; `TryAddEnumerable` plus MS.DI 8 keyed services let you hand-roll decoration without a dependency when the need is small.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016 | First NuGet release; `Scan` assembly-scanning API (repo created 2015-11)[^3]. |
| 3.x | ~2019 | Decoration API matured; open-generic support. |
| 4.x | ~2021 | API refinements, netstandard/modern TFM alignment. |
| 5.x | 2023–2024 | .NET 8 support, keyed-service-aware decoration. |

Exact minor-version dates are approximate; consult the NuGet version history for authoritative release timing[^4].

## References

[^1]: Scrutor README — Scan / Decorate usage, compiler-generated type opt-in. https://github.com/khellang/Scrutor/blob/master/README.md
[^2]: Microsoft docs, "Dependency injection in .NET" — MS.DI as a minimal container, third-party container guidance. https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection
[^3]: khellang/Scrutor repository metadata — created 2015-11-14. https://github.com/khellang/Scrutor
[^4]: Scrutor on NuGet — release/version history. https://www.nuget.org/packages/Scrutor

## Tags

csharp, dotnet, dependency-injection, ioc-container, assembly-scanning, decorator-pattern, aspnet-core, microsoft-extensions, convention-registration, nuget
