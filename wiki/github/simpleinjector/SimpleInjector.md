# simpleinjector/SimpleInjector

> A deliberately strict .NET dependency-injection container that trades permissiveness for compile-time-like verification of your object graph.

[GitHub repo](https://github.com/simpleinjector/SimpleInjector) ·
[Official website](https://simpleinjector.org) ·
[License: MIT](https://github.com/simpleinjector/SimpleInjector/blob/master/LICENSE.txt)

## Overview

Simple Injector is an Inversion-of-Control container for .NET, maintained
primarily by Steven van Deursen, co-author (with Mark Seemann) of *Dependency
Injection Principles, Practices, and Patterns*[^1]. The project predates its
GitHub home — it began on CodePlex around 2010 and moved to GitHub in 2015[^2].
Its stated goal is to steer developers "toward the pit of success": the API is
small, and the container refuses configurations it considers dangerous rather
than silently accepting them.

The defining tension is opinionatedness versus convenience. Where most
containers try to be permissive drop-ins that resolve whatever you ask, Simple
Injector treats a number of common DI patterns as mistakes and makes them hard
or impossible: it discourages property injection, has no first-class "named
registration," locks itself immutable after the first resolve, and — through its
`Verify()` and diagnostic APIs — actively reports lifestyle mismatches and
captive dependencies that other containers let through. This is a feature for
teams practicing strict SOLID design and a friction point for teams who want a
container that just gets out of the way.

Its second defining trait is speed. Registrations compile to expression trees on
first use, so steady-state resolution is close to hand-written `new` code, and
Simple Injector consistently places among the fastest containers in community
benchmarks[^3].

## Getting Started

```bash
dotnet add package SimpleInjector
```

```csharp
using SimpleInjector;

var container = new Container();

// Register: interface -> implementation, with an explicit lifestyle
container.Register<IUserRepository, SqlUserRepository>(Lifestyle.Transient);
container.Register<ILogger, MailLogger>(Lifestyle.Singleton);
container.Register<UserController>();

// Verify the whole object graph at startup — throws on misconfiguration
container.Verify();

// Resolve (typically done by framework integration, not by hand)
var controller = container.GetInstance<UserController>();
```

`Verify()` walks every registration, builds each object graph, and runs the
diagnostic analyzers. Calling it at startup turns most DI mistakes into a
fail-fast at boot instead of a `NullReferenceException` deep in a request.

## Architecture / How It Works

Registration is explicit and lifestyle-typed. The three built-in lifestyles are
`Transient` (new instance per resolve), `Singleton` (one per container), and
`Scoped` — the latter realized through pluggable implementations such as
`AsyncScopedLifestyle` (async/`IAsyncDisposable`-aware) and
`ThreadScopedLifestyle`, plus per-request scopes supplied by the integration
packages[^4].

Under the hood each registration is turned into an `Expression` and compiled to a
delegate on first resolution. Deep graphs are built in full — the container walks
constructor parameters recursively — and constructor injection is the only
first-class injection mode. Property injection exists but must be opted into via a
custom `IPropertySelectionBehavior`; this is intentional friction, because the
maintainers regard property injection as a code smell.

Key opinionated behaviors:

- **Immutability after first use.** Once the first instance is resolved the
  container locks; further registration throws. This prevents "register at
  runtime" service-locator patterns and makes the configuration a build-time
  artifact.
- **No ambiguous registrations.** Re-registering the same service type throws
  rather than silently overriding. There is no built-in "named registration";
  the recommended replacement is conditional/contextual registration or
  collection registration (`Container.Collection.Register`).
- **Auto-resolution of unregistered concrete types.** By default the container
  will construct an unregistered concrete class if all its dependencies resolve.
  This is convenient but a genuine footgun — it can mask a missing registration —
  and can be disabled via `Options.ResolveUnregisteredConcreteTypes`.
- **Diagnostics.** `Verify()` and `Container.GetCurrentRegistrations()` /
  `Analyzer.Analyze()` surface warnings for lifestyle mismatches (a.k.a. captive
  dependencies — e.g. a `Transient` injected into a `Singleton`), torn
  lifestyles, short-circuited dependencies, and disposable transients[^5].

Framework support is delivered through separate `SimpleInjector.Integration.*`
NuGet packages (ASP.NET Core MVC, Web API, WCF, and others). The container
targets .NET Framework 4.5+ and .NET Standard 2.0, so it runs on modern .NET
(Core, 5+), Mono, and Xamarin.

## Production Notes

- **ASP.NET Core integration is a bridge, not a takeover.** Simple Injector does
  not replace `Microsoft.Extensions.DependencyInjection`; the framework's own
  services stay in the MS container while your application components live in
  Simple Injector, with "cross-wiring" between them[^6]. This is deliberate —
  the MS container permits things Simple Injector rejects — but it means the
  wiring is more verbose than a drop-in replacement, and every tutorial that
  assumes `IServiceCollection` needs translation.
- **Treat `Verify()` as a required CI gate.** Its value is mostly lost if it runs
  only on a developer's machine. Run it in a test so lifestyle mismatches and
  captive dependencies fail the build, not production.
- **Captive dependencies are the classic bug.** Injecting a `Scoped` (e.g. a
  DbContext) into a `Singleton` silently pins one scoped instance for the app
  lifetime. Simple Injector flags this by default — but only if you actually call
  `Verify()`.
- **Immutability bites late binding.** Plugins or modules that want to register
  services after the app has started will hit the "container is locked" wall.
  Register everything before the first resolve; use factories or
  `Container.Collection` for dynamic sets.
- **Disposal is scope-bound.** Transient disposables are not tracked/disposed by
  the container by design (it would create leaks); dispose them yourself or make
  them scoped. This surprises developers coming from containers that auto-track.
- **Upgrade friction is low but real.** The major-version jumps (v3 → v4 → v5)
  dropped legacy target frameworks and tightened defaults; most breakage on
  upgrade comes from previously-tolerated registrations now being rejected.

## When to Use / When Not

**Use when:**

- You practice strict constructor-injection SOLID design and want the container
  to enforce it.
- You value fail-fast startup verification and lifestyle-mismatch diagnostics.
- You want near-hand-written resolution performance.
- Your composition root is static and known at startup.

**Avoid when:**

- You want a zero-friction default for ASP.NET Core — the built-in MS container
  or a drop-in-compatible one is less work.
- You depend on named registrations, property injection, or heavy runtime
  re-registration.
- You need built-in dynamic-proxy interception (Simple Injector supports it only
  via decorators / third-party proxies).
- Your team treats the container as a general-purpose service locator.

## Alternatives

- autofac/Autofac — feature-rich and permissive, with modules and assembly
  scanning; use it when you want flexibility Simple Injector deliberately
  withholds.
- dotnet/runtime (Microsoft.Extensions.DependencyInjection) — the built-in
  ASP.NET Core container; use it when you want the framework-default happy path
  with no bridging.
- castleproject/Windsor — mature container with first-class dynamic interception;
  use it when AOP/proxying is central.
- JasperFx/lamar — fast, MS-DI-compatible successor to StructureMap; use it when
  you want StructureMap ergonomics plus drop-in `IServiceProvider` compatibility.
- dadhi/DryIoc — very fast micro-container with a large feature surface; use it
  when raw speed and configurability outweigh a small API.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | ~2010 | Originated on CodePlex[^2]. |
| 3.0 | 2015 | GitHub-era baseline; broad .NET Framework support[^2]. |
| 4.0 | 2017 | Dropped legacy targets; API cleanups. |
| 5.0 | 2020-07 | Major release: revised collection APIs, metadata, ambient scoping, tightened defaults[^7]. |
| 5.x | 2020–2026 | Ongoing maintenance releases on the v5 line. |

## References

[^1]: Steven van Deursen & Mark Seemann, *Dependency Injection Principles, Practices, and Patterns*, Manning, 2019. https://www.manning.com/books/dependency-injection-principles-practices-patterns
[^2]: Simple Injector project home and release history. https://simpleinjector.org and https://github.com/simpleinjector/SimpleInjector/releases
[^3]: DI container benchmark comparisons (community-maintained). https://danielpalme.github.io/IocPerformance/
[^4]: Simple Injector documentation, "Object Lifetime Management." https://simpleinjector.org/lifetimes
[^5]: Simple Injector documentation, "Diagnostic Services." https://simpleinjector.org/diagnostics
[^6]: Simple Injector documentation, "ASP.NET Core Integration." https://simpleinjector.org/aspnetcore
[^7]: Simple Injector documentation, "Simple Injector v5.x Migration Guide." https://simpleinjector.org/migration

## Tags

csharp, dotnet, dependency-injection, ioc-container, di-container, constructor-injection, solid, container-verification, aspnet-core, nuget
