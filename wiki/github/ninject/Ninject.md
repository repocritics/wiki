# ninject/Ninject

> A dependency-injection container for .NET, built around a fluent binding DSL and contextual binding — now a legacy option in a world where .NET ships its own container.

[GitHub repo](https://github.com/ninject/Ninject) ·
[Official website](http://ninject.org/) ·
License: Apache-2.0 OR Ms-PL (dual)[^1]

## Overview

Ninject is one of the oldest inversion-of-control (IoC) containers for .NET, created by Nate Kohari and first released around 2008[^2]. Its pitch was a code-first configuration model — a fluent, type-safe binding DSL (`Bind<IWeapon>().To<Sword>()`) at a time when most .NET containers were configured through XML files and string identifiers. That design choice aged well; XML-configured containers largely disappeared and every modern .NET container now configures in code.

Ninject's other defining feature was **contextual binding**: the ability to resolve a different concrete implementation of a service depending on *where* it is being injected (which type is asking, what it was named, what constraints apply). The README claims Ninject was the first DI container to support this. It is expressive but is also the part of the container most prone to surprising resolution behavior at scale.

The central tension for anyone evaluating Ninject in 2026 is that .NET Core and .NET 5+ ship `Microsoft.Extensions.DependencyInjection` (MS.DI) as a built-in container wired into the framework's hosting, configuration, and ASP.NET Core request pipeline[^3]. Ninject predates that by roughly a decade, and much of what made it attractive — a code-first container with lifetime scopes and constructor injection — is now table stakes provided out of the box. Ninject is still maintained (the repository saw commits in 2024), but its role has shifted from "the DI container you reach for" to "a container you keep because an existing codebase already depends on it."

## Getting Started

```bash
dotnet add package Ninject
```

```csharp
using Ninject;
using Ninject.Modules;

public interface IWeapon { }
public class Sword : IWeapon { }

public class Samurai
{
    public IWeapon Weapon { get; }
    public Samurai(IWeapon weapon) => Weapon = weapon;
}

public class WarriorModule : NinjectModule
{
    public override void Load() => Bind<IWeapon>().To<Sword>();
}

class Program
{
    static void Main()
    {
        var kernel = new StandardKernel(new WarriorModule());
        var samurai = kernel.Get<Samurai>();   // Sword injected via constructor
    }
}
```

The `IKernel` (`StandardKernel`) is the composition root. Bindings live in `NinjectModule` subclasses or are declared inline against the kernel. Lifetime is controlled by fluent scope calls: `.InSingletonScope()`, `.InTransientScope()` (the default), `.InThreadScope()`, and `.InRequestScope()` (via the web extension).

## Architecture / How It Works

Ninject resolves a request by walking an **activation pipeline**. Given a requested type and context, it selects a matching binding, chooses a constructor (by default the one with the most parameters it can satisfy), recursively resolves each parameter, constructs the instance, and then runs activation strategies (property injection, method injection, `IInitializable`/`IStartable` hooks) before handing it back.

Rather than calling constructors purely through `System.Reflection` invocation, Ninject uses lightweight code generation — it emits IL via `DynamicMethod`/expression-based injectors to build activation delegates. The README advertises this as an 8–50x speedup over naive reflection invocation. That claim is about Ninject-versus-reflection, not Ninject-versus-other-containers; in cross-container microbenchmarks Ninject has historically sat at the *slow* end of the field, because its per-resolution pipeline (context construction, binding resolution, planning, activation strategies) does more work than leaner containers[^4]. For the vast majority of applications this cost is irrelevant — resolution happens at composition time, not in hot loops — but it undercuts the "fast" framing when the comparison is to peers rather than to reflection.

The core deliberately ships thin: a single assembly with no dependencies outside the base class library. Anything beyond core constructor/property injection lives in separately versioned **extension packages** (`Ninject.Extensions.*`) — factory generation, interception/AOP, convention-based binding, child kernels, XML config, and the web/MVC/WebApi/OWIN integration packages. This keeps the core small but means real-world setups pull in several independently maintained extensions, each with its own version-compatibility surface against the core.

Contextual binding is implemented through `When(...)` conditions and named bindings evaluated against the `IRequest`/`IContext` at resolve time. Constraints like `WhenInjectedInto<T>()`, `WhenParentNamed(...)`, and generic `When(ctx => ...)` predicates let one interface map to many implementations. This is powerful but shifts wiring logic into predicates that are only exercised at runtime, which is where hard-to-diagnose "wrong implementation injected" bugs come from.

## Production Notes

- **Resolution failures are runtime, not compile-time.** A missing or ambiguous binding throws `ActivationException` when the object is first requested, not when the app builds. Contextual/conditional bindings widen the gap between "compiles" and "resolves correctly." Treat the composition root as something to smoke-test on startup.

- **Captive dependencies.** As with every scoped container, injecting a shorter-lived service (per-request) into a singleton captures it for the singleton's lifetime. Ninject does not diagnose this the way MS.DI's scope-validation does; you have to catch it in review.

- **Property injection is available but easy to overuse.** `[Inject]` on properties/fields works, but hidden property injection makes dependencies invisible at the constructor and complicates testing. Prefer constructor injection; reserve property injection for genuinely optional dependencies.

- **Extension version drift.** Because features are spread across `Ninject.Extensions.*` packages on their own release cadences, upgrading the core can leave extensions behind (or vice versa). Pin and upgrade the set together; mismatched versions surface as binding or activation errors rather than clean load failures.

- **Framework integration is fading.** Ninject's ASP.NET MVC/WebApi/OWIN integration targeted the classic .NET Framework request pipeline. On ASP.NET Core, the idiomatic path is MS.DI, and bridging Ninject in requires `Ninject.Extensions.Microsoft.DependencyInjection` (or hand-wiring an `IServiceProvider`) — a supported but second-class integration compared to a native MS.DI setup.

- **Maintenance cadence.** The project is not abandoned — the default branch saw activity in 2024 — but releases are infrequent and the container is effectively feature-stable. Do not expect it to track new .NET DI conventions quickly. For a greenfield project this is a reason to prefer the built-in container or a more actively developed third-party one.

## When to Use / When Not

**Use when:**
- You maintain an existing codebase already built on Ninject and rewiring to another container is not worth the churn.
- You specifically need Ninject's contextual/conditional binding expressiveness and are comfortable with runtime-resolved wiring.
- You're on classic .NET Framework where there is no built-in container and Ninject's mature integration packages still apply.

**Avoid when:**
- You're starting a new .NET Core / .NET 5+ project — `Microsoft.Extensions.DependencyInjection` is built in, framework-integrated, and sufficient for most apps.
- Raw resolution throughput matters and you're benchmarking against peers; leaner containers resolve faster.
- You want a container that actively tracks current .NET DI conventions and ships frequent releases.

## Alternatives

- dotnet/runtime (Microsoft.Extensions.DependencyInjection) — use instead when you're on modern .NET and want the built-in, framework-integrated default.
- autofac/Autofac — use instead when you want a widely adopted, actively maintained third-party container with rich features and good ASP.NET Core integration.
- simpleinjector/SimpleInjector — use instead when you want strict correctness (verifiable configuration, captive-dependency diagnostics) and high performance.
- castleproject/Windsor — use instead when you need mature interception/AOP and a long-standing feature-rich container.
- JasperFx/lamar — use instead when you want a fast MS.DI-compatible container (StructureMap's successor) as a drop-in replacement.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | ~2008 | Initial release; fluent code-first binding DSL, contextual binding[^2]. |
| 2.0 | ~2011 | Rework of the activation/scoping model. |
| 3.0 | ~2012 | Source moved to git/GitHub; core kept thin, features pushed into `Ninject.Extensions.*`[^5]. |
| 3.2 | ~2014 | Continued 3.x line; extension ecosystem maturity. |
| 3.3.x | later 3.x | .NET Standard 2.0 support; the current maintenance line on NuGet. |

## References

[^1]: Ninject README — "Ninject is dual-licensed... under either the Apache License, Version 2.0, or the Microsoft Public License (Ms-PL)." GitHub reports the license as NOASSERTION because the dual license is not machine-classified. https://github.com/ninject/Ninject/blob/main/LICENSE.txt
[^2]: Ninject project site and README, authored by Nate Kohari. https://github.com/ninject/Ninject#readme
[^3]: .NET docs, "Dependency injection in .NET" — the built-in `Microsoft.Extensions.DependencyInjection` container. https://learn.microsoft.com/dotnet/core/extensions/dependency-injection
[^4]: Daniel Palme, ".NET IoC Container benchmark" — recurring cross-container performance comparisons in which Ninject consistently ranks among the slowest. https://github.com/danielpalme/IocPerformance
[^5]: Ninject wiki, "Changes in Ninject 3." https://github.com/ninject/ninject/wiki/Changes-in-Ninject-3

## Tags

csharp, dotnet, dependency-injection, ioc-container, inversion-of-control, di, framework, legacy, contextual-binding, composition-root
