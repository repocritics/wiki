# reactiveui/splat

> A grab-bag of cross-platform .NET primitives — service location, logging, drawing, and mode detection — that grew out of and underpins ReactiveUI.

[GitHub repo](https://github.com/reactiveui/splat) ·
[NuGet](https://www.nuget.org/packages/Splat) ·
[License: MIT](https://github.com/reactiveui/splat/blob/main/LICENSE)

## Overview

Splat is a .NET utility library maintained by the ReactiveUI team. It bundles a
set of things that are "basically impossible to do in cross-platform mobile
code today, yet there's no reason why": a service locator, a logging facade,
cross-platform image loading/saving, a port of `System.Drawing.Color` and
geometry primitives (`PointF`, `SizeF`, `RectangleF`) for portable libraries,
and helpers to detect whether code is running in a unit-test runner or XAML
design mode.[^1] The name and scope are deliberately unglamorous — it exists to
absorb the `#ifdef`-riddled platform glue that every UI app accumulates.

The library is best understood as ReactiveUI's foundation rather than a
standalone product. `Locator.Current` / `AppLocator.Current` is the service
locator that ReactiveUI itself resolves views, view models, and schedulers
through, and most Splat installs arrive transitively as a ReactiveUI
dependency.[^2] Adopting Splat directly means opting into the service-locator
pattern — a global, static, ambient resolver that the wider .NET DI community
generally treats as an anti-pattern. Splat's answer is a set of adapter
packages (Autofac, DryIoc, Microsoft.Extensions.DependencyInjection, Ninject,
SimpleInjector) that let you back `Locator.Current` with a real container, so
the static surface becomes a shim over your DI framework of choice.[^3]

The defining tension is exactly that: Splat is a genuinely useful cross-platform
toolbox, but its central abstraction is a global service locator, and its
gravity is toward the ReactiveUI ecosystem. Teams that want plain constructor
injection with `Microsoft.Extensions.DependencyInjection` will find Splat's core
value proposition (the locator) redundant, and will pull it in only because
ReactiveUI requires it.

## Getting Started

```bash
dotnet add package Splat
# Optional: cross-platform drawing/image loading lives in Splat.Drawing
dotnet add package Splat.Drawing
```

```cs
using Splat;

// Register at application startup.
Locator.CurrentMutable.Register<IToaster>(() => new Toaster());          // transient
Locator.CurrentMutable.RegisterConstant<IConfig>(myConfig);              // singleton
Locator.CurrentMutable.RegisterLazySingleton<ILogger>(() => new FileLogger());

// Resolve anywhere.
var toaster = Locator.Current.GetService<IToaster>();
var plugins = Locator.Current.GetServices<IPlugin>();   // all registrations

// Logging facade — implement IEnableLogger, then:
this.Log().Info("started");

// Guard test-only / design-time code paths.
if (ModeDetector.InUnitTestRunner()) { /* ... */ }
```

## Architecture / How It Works

Splat is split across a small `Splat` core (locator, logging, mode detection)
and `Splat.Drawing` (colors, geometry, bitmap loading), plus per-container
adapter packages and per-logger adapter packages (Serilog, NLog,
Microsoft.Extensions.Logging, Log4Net). It multi-targets a wide matrix — .NET
Framework 4.6.2/4.7.2, .NET Standard 2.0, .NET 6, and .NET 8 — and supports WPF,
WinForms, WinUI 3, MAUI, and Avalonia.[^1]

The service locator has two faces: `AppLocator.Current` (formerly
`Locator.Current`) for retrieval and `AppLocator.CurrentMutable` for
registration. Both point at a single swappable `IDependencyResolver`. Since v19
the default resolver is no longer the reflection-heavy `ModernDependencyResolver`
but one of two AOT-oriented implementations: `GlobalGenericFirstDependencyResolver`
(process-wide static generic containers) and the now-default
`InstanceGenericFirstDependencyResolver` (per-resolver isolation via a
`ConditionalWeakTable`).[^4] Both use a generic-first design: `GetService<T>()`
hits a static `Container<T>` with no dictionary lookup, while the
`GetService(Type)` overload falls back to a `ConcurrentDictionary` for interop
and dynamic scenarios. Splat documents the older `ModernDependencyResolver` as
having O(n²) registration growth and a global `ReaderWriterLockSlim`; the
GenericFirst resolvers claim O(1) registration and lock-free reads.[^4]

`ModeDetector` and `PlatformModeDetector` inspect the runtime to decide whether
you are inside a test host or a XAML designer — used pervasively so that library
code can avoid touching platform state at design time. The drawing layer's
"leaky abstraction" is intentional: `ToNative()` / `FromNative()` extension
methods convert Splat's portable bitmap and color types into the platform
representation, so you load an image in shared code and materialize it in the
view.[^1]

## Production Notes

- **You are adopting a global service locator.** `Locator.Current` is static
  ambient state. In tests this is a shared-mutable-state footgun: registrations
  leak across tests unless you isolate them. Splat's own guidance is to use
  `InstanceGenericFirstDependencyResolver` for tests and library code, and to
  reserve the Global resolver for single-owner application startup.[^4]

- **Resolver semantics differ by adapter, and "replace vs append" matters.**
  Different backing containers handle duplicate registrations differently; Splat
  warns that a container which appends multiple registrations can produce
  "undesired behaviours, such as the wrong logger factory being used." Read the
  specific adapter README before wiring one in.[^3]

- **ReactiveUI initialization ordering.** When overriding Splat's defaults with
  a custom container, ReactiveUI must be initialized before the container
  finalizes, or its registrations are lost. This ordering bug is a recurring
  support question.[^3]

- **Prefer generic over Type-based APIs.** `GetService<T>()` is lock-free and
  allocation-free on the hot path; `GetService(typeof(T))` takes a dictionary
  lookup and may box. Splat's published micro-benchmarks put generic resolution
  around 50–100 ns versus 200–500 ns for the Type-based path.[^4] Treat these as
  the maintainer's internal numbers, not independently verified.

- **AOT / trimming.** Only v19+ is meaningfully NativeAOT-friendly. Reflection-
  free configuration goes through the `AppBuilder` + `IModule` pattern rather
  than the classic reflection-based registration. Older versions and the legacy
  resolver were poor AOT/trimming citizens.[^4]

- **The default resolver changed under you.** Moving from v18 to v19+ swaps the
  default resolver implementation. It shares the `IMutableDependencyResolver`
  interface so recompilation is usually enough, but behavior around global vs
  isolated container state and `Clear()` teardown semantics differs — audit
  test cleanup after upgrading.[^4]

## When to Use / When Not

**Use when:**
- You are building on ReactiveUI — Splat is not optional, it is the substrate.
- You want one small dependency for cross-platform color/geometry/bitmap types
  and a test/design-mode detector shared across WPF, MAUI, Avalonia, and WinUI.
- You want a service-locator seam you can later back with a real container.

**Avoid when:**
- You are doing idiomatic constructor injection with
  `Microsoft.Extensions.DependencyInjection` and have no ReactiveUI — the
  locator adds ambient global state you do not need.
- You are AOT-constrained and stuck on a pre-v19 version.
- You want a single-purpose logging or DI library; Splat's grab-bag scope means
  you pull in more surface than you use.

## Alternatives

- dotnet/runtime (Microsoft.Extensions.DependencyInjection) — use instead when you want the standard .NET DI container with constructor injection and no service-locator pattern.
- autofac/Autofac — use instead when you need advanced container features (scoping, decorators, modules) as the primary DI story.
- serilog/serilog — use instead when logging is the actual need; Splat's logging is a thin facade you would bridge to Serilog anyway.
- reactiveui/ReactiveUI — not an alternative but the reason most people have Splat; evaluate them together.
- xamarin/Xamarin.Forms era helpers / MAUI Essentials — use instead when you only need platform-detection and device primitives, not a locator.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-06-30 | Repo created; utilities extracted from the ReactiveUI ecosystem.[^2] |
| 14.x | 2022–2023 | Long-lived line across the .NET 6/7 era; adapter packages matured. |
| 15.x | 2024-05 → 2025-08 | Extended multi-target support (.NET 8), most-deployed line for two years. |
| 19.x | 2026-01 | GenericFirst AOT resolvers introduced; `ModernDependencyResolver` demoted to compatibility.[^4] |
| 20.0.0 | 2026-06-12 | Current major release.[^5] |

As of 2026-07 the project is actively maintained (last push 2026-07-17), sitting
just under 1,000 stars with ~140 forks — modest raw numbers that understate its
reach, since it ships transitively under ReactiveUI's much larger install base.

## References

[^1]: Splat README — features, install matrix, drawing/`ToNative()` abstraction, `ModeDetector`. https://github.com/reactiveui/splat/blob/main/README.md
[^2]: ReactiveUI project — Splat is its dependency-resolution and utility foundation. https://github.com/reactiveui/ReactiveUI
[^3]: Splat README — dependency-resolver adapter packages (Autofac, DryIoc, Microsoft.Extensions.DependencyInjection, Ninject, SimpleInjector) and replace-vs-append / ReactiveUI init warnings. https://github.com/reactiveui/splat#dependency-resolver-packages
[^4]: Splat README — GenericFirst resolvers, container architecture, benchmarks, and the ModernDependencyResolver comparison (v19+). https://github.com/reactiveui/splat#the-default-dependency-resolver-v19
[^5]: Splat releases — v20.0.0 published 2026-06-12. https://github.com/reactiveui/splat/releases

## Tags

csharp, dotnet, service-locator, dependency-injection, cross-platform, logging, reactiveui, ioc, maui, aot
