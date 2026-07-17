# autofac/Autofac

> A mature, feature-rich inversion-of-control container for .NET, known for lifetime scopes, modules, and implicit relationship types.

[GitHub repo](https://github.com/autofac/Autofac) ·
[Official website](https://autofac.org) ·
[License: MIT](https://github.com/autofac/Autofac/blob/develop/LICENSE)

## Overview

Autofac is a dependency-injection (IoC) container for .NET. It resolves the object graph of an application from registrations you declare up front, managing object lifetimes and disposal along the way. It originated in the late 2000s — created by Nicholas Blumhardt — and long predates the current GitHub repository, which dates to a 2014 migration[^1]. It remains one of the most widely used third-party containers in the .NET ecosystem alongside the framework's own `Microsoft.Extensions.DependencyInjection`.

The defining tension around Autofac is exactly that comparison. Since ASP.NET Core shipped a built-in DI container in 2016, the baseline question for any .NET project is no longer "which container" but "do I need more than the built-in one." Autofac's answer is a set of features the built-in container deliberately omits: assembly scanning, registration modules, decorators and adapters, property injection, aggregate services, multitenancy, and a family of "implicit relationship types" (`Func<T>`, `Lazy<T>`, `Owned<T>`, `Meta<T>`, `IEnumerable<T>`, `IIndex<K,V>`) that let consumers express lifetime and construction intent through the type they depend on. If you need none of those, the built-in container is the lower-friction choice; Autofac earns its place when you do.

The project is mature and quietly maintained rather than fast-moving — the low open-issue count (single digits) against 4.6k stars reflects a stable, well-triaged codebase, not abandonment; commits land regularly on the `develop` branch. The core `Autofac` package sits at the center of a large constellation of integration packages (ASP.NET MVC/Web API, OWIN, SignalR, WCF, Service Fabric, configuration, multitenancy, DynamicProxy interception) maintained under the same organization.

## Getting Started

Autofac is distributed on NuGet[^2]:

```bash
dotnet add package Autofac
```

Register components against a `ContainerBuilder`, build the container, then resolve from a lifetime scope:

```csharp
using Autofac;

var builder = new ContainerBuilder();

builder.RegisterType<TaskRepository>().As<ITaskRepository>();
builder.RegisterType<TaskController>();

// Assembly scanning: register every type ending in "Service".
builder.RegisterAssemblyTypes(typeof(Program).Assembly)
       .Where(t => t.Name.EndsWith("Service"))
       .AsImplementedInterfaces();

var container = builder.Build();

// Resolve from a nested scope, not the root container, so
// scoped instances are disposed when the scope ends.
using (var scope = container.BeginLifetimeScope())
{
    var controller = scope.Resolve<TaskController>();
}
```

For ASP.NET Core, wire Autofac in as the service-provider factory via the `Autofac.Extensions.DependencyInjection` package rather than using the core package directly[^3].

## Architecture / How It Works

Registration and resolution are separated into two phases. A `ContainerBuilder` accumulates registrations; `Build()` produces an immutable `IContainer`. Each registration maps one or more exposed services to a component with an activator (reflection-based type activation, a delegate, or a pre-built instance) and a lifetime.

**Lifetime scopes** are the central concept. The container is the root scope; `BeginLifetimeScope()` creates nested child scopes. Each registration has a sharing model:

- `InstancePerDependency` (the default — a new instance every resolve; equivalent to "transient").
- `SingleInstance` (one shared instance for the container's lifetime).
- `InstancePerLifetimeScope` (one instance per scope; equivalent to "scoped").
- `InstancePerMatchingLifetimeScope` / `InstancePerRequest` (tagged scopes, e.g. one per web request).

**Disposal is tracked automatically.** Any resolved component implementing `IDisposable` is registered with the scope that created it and disposed when that scope is disposed. This is convenient and also the source of Autofac's most common production bug (see below).

**Implicit relationship types** are resolved without extra registration. Depending on `Func<T>` gives you a factory; `Lazy<T>` defers construction; `Owned<T>` hands you an instance whose lifetime you dispose explicitly; `IEnumerable<T>` collects every registration of `T`; `Meta<T>` and `IIndex<K,V>` expose registration metadata.

**Modules** (`Autofac.Module` subclasses) package related registrations behind an `override void Load(ContainerBuilder)`, giving a unit of composition that can be reused and conditionally configured.

Autofac 6.0 (2020) rewrote resolution around a **resolve pipeline** built from ordered middleware, replacing the older activation model. Decorators, interception (via `Autofac.Extras.DynamicProxy`), and diagnostics hook into that pipeline rather than into ad-hoc extension points[^4].

## Production Notes

**The captive-dependency / disposal-leak footgun.** Because Autofac tracks every resolved `IDisposable` against the resolving scope, resolving disposable components directly from the root container (instead of a child scope) means those instances are held — and never disposed — for the whole application lifetime. In a long-running server this presents as a slow memory leak. The fix is disciplinary: resolve from per-request/per-operation scopes, and never resolve transient disposables from the container root. Autofac emits diagnostic warnings for some of these cases but does not prevent them.

**No build-time graph validation by default.** Unlike the built-in container's `ValidateOnBuild`, a missing or ambiguous registration in Autofac surfaces as a `DependencyResolutionException` at resolve time, not at `Build()`. Teams that want fail-fast startup should add integration/smoke tests that resolve their root graph, or use the diagnostics tooling.

**Constructor selection can surprise you.** By default Autofac picks the greediest constructor whose parameters it can all resolve. Adding an optional dependency, or a new constructor overload, can silently change which constructor is chosen. Use `UsingConstructor(...)` to pin it when a type has multiple constructors.

**Performance.** Autofac's reflection- and pipeline-based resolution is fast enough for typical web workloads but is not the fastest container available; micro-benchmark-sensitive workloads sometimes reach for DryIoc, Grace, or the built-in container. Register `SingleInstance` where semantics allow, and prefer resolving whole graphs once per operation over resolving in tight loops.

**Upgrade friction.** The 6.0 pipeline rewrite changed several extensibility internals; libraries that hooked deep into activation needed updates. Major versions have also periodically dropped end-of-life target frameworks, so pinned older TFMs can block an upgrade. The public registration API (`ContainerBuilder`, lifetime methods) has stayed broadly stable across versions.

**ASP.NET Core integration is indirect.** You keep populating the framework `IServiceCollection`, then hand Autofac control via `UseServiceProviderFactory(new AutofacServiceProviderFactory())` and a `ConfigureContainer` callback. Registrations made both ways coexist, which is powerful but means two registration surfaces to reason about.

## When to Use / When Not

**Use when:**
- You need features the built-in container omits: assembly scanning, modules, decorators/adapters, property injection, aggregate services, or multitenancy.
- You rely on relationship types (`Func<T>`, `Owned<T>`, `Lazy<T>`, `IEnumerable<T>`) to express construction and lifetime intent at the consumer.
- You are integrating a range of frameworks (MVC, Web API, OWIN, SignalR, WCF) and want one container with first-party integration packages.

**Avoid when:**
- Your composition is simple and the built-in `Microsoft.Extensions.DependencyInjection` already covers it — an extra dependency buys nothing.
- You need maximum resolve throughput and are willing to trade features for it (consider DryIoc or Grace).
- You want compile-time / build-time verification of the graph as a hard guarantee (SimpleInjector's diagnostics or source-generated DI fit that better).

## Alternatives

- microsoft/dotnet (`Microsoft.Extensions.DependencyInjection`) — use the built-in container when you don't need Autofac's advanced features; it added keyed services in .NET 8.
- dadhi/DryIoc — use when raw resolution performance is the priority and you accept a steeper API.
- ipjohnson/Grace — use when you want a fast container with rich features and expression-compiled resolution.
- simpleinjector/SimpleInjector — use when you want strict, opinionated verification and diagnostics that catch misconfiguration early.
- castleproject/Windsor — use when interception/AOP and a long-established feature set are central to your design.

## History

| Version | Date | Notes |
|---------|------|-------|
| (origin) | ~2007–2008 | Created by Nicholas Blumhardt; hosted outside GitHub initially[^1]. |
| GitHub migration | 2014 | Repository created on GitHub[^1]. |
| 4.0 | 2016 | .NET Standard / .NET Core support; broad retargeting[^4]. |
| 5.0 | 2020 | Nullable reference types, target-framework cleanup[^4]. |
| 6.0 | 2020 | Resolve-pipeline (middleware) rewrite; new decorator/interception model[^4]. |
| 7.0 | 2023 | Continued modernization and TFM updates[^4]. |
| 8.0 | 2024 | Further framework retargeting and refinements[^4]. |

## References

[^1]: autofac/Autofac repository metadata (GitHub repo created 2014-01-22) and project background. https://github.com/autofac/Autofac
[^2]: Autofac on NuGet. https://www.nuget.org/packages/Autofac/
[^3]: Autofac ASP.NET Core integration documentation. https://autofac.readthedocs.io/en/latest/integration/aspnetcore.html
[^4]: Autofac release notes and documentation. https://github.com/autofac/Autofac/releases and https://autofac.readthedocs.io/

## Tags

csharp, dotnet, dependency-injection, ioc-container, inversion-of-control, netcore, netstandard, lifetime-scopes, library
