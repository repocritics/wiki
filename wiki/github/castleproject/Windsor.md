# castleproject/Windsor

> A mature, feature-heavy Inversion of Control container for .NET, built around the Castle MicroKernel and DynamicProxy interception.

[GitHub repo](https://github.com/castleproject/Windsor) ·
[Official website](http://www.castleproject.org) ·
[License: Apache-2.0](https://github.com/castleproject/Windsor/blob/master/LICENSE)

## Overview

Castle Windsor is one of the oldest dependency-injection containers in the .NET ecosystem, with roots in the Castle Project going back to roughly 2004[^1]. It predates Microsoft's own `Microsoft.Extensions.DependencyInjection` (MS.DI) by over a decade, and it grew up in the era when a container was expected to do much more than construct object graphs — interception/AOP, typed factories, lifecycle decommission, XML configuration, and pluggable "facilities" are all first-class here.

That heritage is both the appeal and the tension. Windsor's feature surface is large and its registration DSL is expressive, but the defaults reflect a 2000s philosophy: property injection is on by default, and the container *tracks* the components it creates so it can dispose them later. Teams coming from the minimal MS.DI model are frequently surprised by both. Windsor rewards operators who learn its lifecycle model and punishes those who treat it as a drop-in constructor-injection container.

As of 2026 the project is stable and still receiving occasional commits, but the release cadence has slowed markedly compared to its 2010–2016 peak[^2]. For most new .NET applications the gravitational default is now MS.DI plus, if needed, Autofac; Windsor's remaining edge is its interception and decommission model.

## Getting Started

```bash
dotnet add package Castle.Windsor
```

```csharp
using Castle.MicroKernel.Registration;
using Castle.Windsor;

var container = new WindsorContainer();

container.Register(
    Component.For<IEmailSender>()
             .ImplementedBy<SmtpEmailSender>()
             .LifestyleTransient());

var sender = container.Resolve<IEmailSender>();
sender.Send("hello");

container.Release(sender);   // required for tracked transient lifecycles
container.Dispose();
```

Real applications organize registration into installers rather than inline calls:

```csharp
public class ServicesInstaller : IWindsorInstaller
{
    public void Install(IWindsorContainer container, IConfigurationStore store)
    {
        container.Register(
            Classes.FromThisAssembly()
                   .BasedOn<IService>()
                   .WithServiceDefaultInterfaces()
                   .LifestyleTransient());
    }
}

// container.Install(FromAssembly.This());
```

## Architecture / How It Works

Windsor is layered on two lower components. The **MicroKernel** is the actual container engine — it holds handlers, lifestyle managers, sub-resolvers, and the release policy. **Windsor** wraps the kernel with the fluent registration DSL, installer discovery, and facility management. Beneath both sits **Castle.Core**, which provides **DynamicProxy** — the runtime proxy generator that powers interception[^3].

Registration is declarative and open-ended: `Component.For<T>()` describes a single service, and `Classes.FromThisAssembly().BasedOn<...>()` describes convention-based batch registration. Lifestyles include Singleton (the default), Transient, Scoped, PerWebRequest, PerThread, Pooled, Bound, and custom implementations.

Two design decisions define the container's behavior more than any feature:

- **Property injection is greedy by default.** Windsor will attempt to satisfy any writable public property whose type it can resolve, unless you opt out. This surprises developers expecting constructor-only injection and can create non-obvious coupling.
- **Component tracking / release policy.** By default Windsor retains a reference to components it creates that have decommission concerns (e.g. `IDisposable`, or components with lifecycle steps) so it can dispose them deterministically when you call `Release`. This is the basis of the **Register–Resolve–Release** discipline documented by longtime maintainer Krzysztof Koźmic[^4].

**Interception** is Windsor's signature capability: an `IInterceptor` can wrap resolved components transparently via DynamicProxy, enabling logging, caching, transactions, and retry policies without altering the target class. **Facilities** (`TypedFactoryFacility`, `StartableFacility`, logging, WCF integration, and others) extend the kernel with cross-cutting registration behavior.

## Production Notes

**The tracked-transient memory leak is the canonical Windsor footgun.** Because the container holds references to tracked transient components until they are released, code that resolves transients in a loop without calling `Release` grows the container's tracked-object graph unbounded. The three standard remedies are: call `Release` on everything you `Resolve` manually, resolve through factories/scopes that release automatically, or install a `NoTrackingReleasePolicy` and manage disposal yourself[^4]. Any long-lived process (services, request handlers) must have an explicit story here.

**Property injection surprises.** Because it is on by default and greedy, a newly added public property can silently start receiving an injected dependency, or a resolution can pull in more of the graph than intended. Many teams add conventions to force constructor injection and ignore properties.

**Performance.** Windsor resolves via runtime reflection and interception rather than compiled expression trees, so raw resolution throughput trails newer containers such as SimpleInjector and MS.DI in microbenchmarks. For typical line-of-business request rates this rarely matters, but it is a real consideration for hot paths that resolve per-call.

**ASP.NET Core integration** is not the native model. Windsor plugs into the generic host via an MS.DI adapter rather than replacing the built-in container; there are known sharp edges where framework services expect MS.DI semantics (open generics, `IEnumerable<T>` resolution, disposal ownership) that differ from Windsor's[^5]. Validate scoped lifetimes and disposal carefully.

**Maintenance cadence.** The project remains open and unarchived, but active development has slowed; verify the latest release and .NET target support against the NuGet feed before adopting for a greenfield app rather than assuming parity with the fast-moving MS.DI/Autofac line.

## When to Use / When Not

**Use when:**
- You need interception/AOP as a core capability and want it integrated with registration rather than bolted on.
- You have an existing Castle Windsor codebase and value continuity over migration churn.
- You want expressive convention-based registration and a rich facility model.
- Deterministic decommission (Release-driven disposal) fits your architecture.

**Avoid when:**
- You are starting a new ASP.NET Core app and want the framework-native path — MS.DI (optionally Autofac) is the lower-friction default.
- You want maximum resolution performance or compile-time verification of the container configuration.
- Your team is unfamiliar with tracked lifecycles and property injection and cannot invest in learning the Register–Resolve–Release model.

## Alternatives

- dotnet/runtime (Microsoft.Extensions.DependencyInjection) — use instead when you want the framework-native, minimal container that every modern .NET library targets.
- autofac/Autofac — use instead when you want a full-featured, actively maintained container with strong ASP.NET Core integration and a large community.
- simpleinjector/SimpleInjector — use instead when you prioritize resolution performance and strict configuration verification at startup.
- JasperFx/lamar — use instead when you want a StructureMap-style registration DSL that is MS.DI-compatible.
- ninject/Ninject — an older convention-driven container of comparable heritage to Windsor, with similarly reduced activity.

## History

| Version | Date | Notes |
|---------|------|-------|
| Castle Project origins | ~2004 | Windsor emerges as the Castle IoC container[^1]. |
| Repo on GitHub | 2011-10-23 | Source migrated to the current GitHub repository. |
| 3.x | ~2011–2014 | Mature-era releases; fluent registration and facilities established. |
| 4.x | ~2016–2017 | Continued .NET Framework support and API refinement. |
| 5.0 | ~2018 | .NET Standard 2.0 / .NET Core support added[^2]. |
| 6.0 | ~2022 | Target-framework modernization; legacy trimming. |

## References

[^1]: Castle Project — project history and overview. http://www.castleproject.org
[^2]: Castle Windsor releases. https://github.com/castleproject/Windsor/releases
[^3]: Castle.Core / DynamicProxy — the interception engine underpinning Windsor. https://github.com/castleproject/Core
[^4]: Krzysztof Koźmic, "Must I release everything when using Windsor?" (Register–Resolve–Release). https://kozmic.net/2010/08/27/must-i-release-everything-when-using-windsor/
[^5]: Castle Windsor documentation. https://github.com/castleproject/Windsor/blob/master/docs/README.md

## Tags

csharp, dotnet, dependency-injection, inversion-of-control, ioc-container, aop, interception, dynamicproxy, castle-project, di-container
