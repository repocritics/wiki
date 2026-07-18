# giraffe-fsharp/Giraffe

> A functional F# micro web framework that composes HTTP handlers on top of the ASP.NET Core pipeline instead of replacing it.

[GitHub repo](https://github.com/giraffe-fsharp/Giraffe) ·
[Official website](https://giraffe.wiki) ·
[License: Apache-2.0](https://github.com/giraffe-fsharp/Giraffe/blob/master/LICENSE)

## Overview

Giraffe is an F# web framework created by Dustin Moris Gorski in January 2017[^1]. Its founding bet, explicitly stated in the README, is the opposite of Suave's: rather than building a standalone functional web server, Giraffe is a thin functional layer that plugs into ASP.NET Core as middleware and reuses Microsoft's hosting, DI, configuration, authentication, and logging stack wholesale. The pitch is "the functional counterpart of ASP.NET Core MVC" — F# idioms on top of the most heavily engineered web runtime in the .NET ecosystem.

Applications are built from `HttpHandler` functions composed with the `>=>` (fish) operator and a `choose` combinator, a design lineage running from Suave's WebParts through Gerard O'Connor's continuation-passing rework that gave Giraffe its Task-based core[^2]. At ~2.2k stars it is a niche framework by absolute numbers, but within the F# ecosystem it is the de facto standard for server-side web work: Saturn is built directly on it, and it anchors most F# web tutorials and templates. The repo moved from a personal account to the `giraffe-fsharp` organization in 2018[^3], and after Gorski stepped back from day-to-day work, maintenance passed to a small volunteer team (64J0, dbrattli, mrtz-j per the current README). Pushes as of July 2026 show the project alive, but release cadence is measured in months, not weeks.

## Getting Started

```bash
dotnet new install "giraffe-template::*"
dotnet new giraffe -o MyApp
# or manually:
dotnet add package Giraffe
```

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection
open Giraffe

let webApp =
    choose [
        route  "/ping"        >=> text "pong"
        routef "/hello/%s"    (fun name -> text $"Hello, {name}")
        route  "/"            >=> htmlFile "pages/index.html"
    ]

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    builder.Services.AddGiraffe() |> ignore
    let app = builder.Build()
    app.UseGiraffe webApp
    app.Run()
    0
```

## Architecture / How It Works

The core abstraction is one type alias:

```fsharp
type HttpFuncResult = Task<HttpContext option>
type HttpFunc       = HttpContext -> HttpFuncResult
type HttpHandler    = HttpFunc -> HttpContext -> HttpFuncResult
```

An `HttpHandler` receives the *next* handler as a continuation and the mutable ASP.NET Core `HttpContext`, and returns `Some ctx` (handled), `None` (skip — lets `choose` try the next branch), or short-circuits. `>=>` is Kleisli composition over this shape; `choose` is ordered alternation. This continuation-passing design replaced an earlier `Async`-bind pipeline for performance and is the reason Giraffe is `Task`-native rather than `Async`-native[^2] — convenient for ASP.NET Core interop, mildly jarring for F# purists.

Two routing modes coexist:

1. **Classic router** — `choose` lists evaluated top-to-bottom per request. Maximally flexible (any handler can decide to match or skip at runtime) but linear-scan: route resolution cost grows with route count, and ASP.NET Core has no knowledge of your routes.
2. **Giraffe.EndpointRouting** — added in 5.0[^4], registers routes into ASP.NET Core's endpoint routing. Faster resolution and, more importantly, required for middleware that inspects endpoint metadata (`UseAuthorization`, CORS policies, rate limiting). The tradeoff: only a subset of combinators works there — arbitrary runtime `choose` logic inside route resolution is not expressible.

`routef` gives format-string typed routes (`%s`, `%i`, `%O` for GUIDs). Model binding (`bindJson`, `bindForm`, `bindQuery`) goes through a pluggable `Json.ISerializer` abstraction; the historic default was Newtonsoft.Json, with `System.Text.Json` the supported modern option — serializer choice changes null/option-field binding behavior, so it is a real migration decision, not a config toggle. `Giraffe.ViewEngine`, an F# DSL that renders typed `XmlNode` trees to HTML, was split into its own NuGet package with 5.0[^4].

Everything else — DI, config, sessions, static files, auth schemes — is deliberately *not* Giraffe's; you use ASP.NET Core's own building blocks. This is the framework's defining tradeoff: minimal surface to maintain, but the docs assume you can read Microsoft's C#-centric documentation and translate.

## Production Notes

- **Middleware ordering is on you.** With the classic router, `app.UseGiraffe webApp` is terminal middleware; `UseAuthentication`, `UseStaticFiles`, etc. must be registered before it, and endpoint-metadata-driven middleware (authorization policies, CORS-per-endpoint) simply won't see your routes unless you adopt EndpointRouting. Silent auth bypass from ordering mistakes is the classic footgun.
- **Route-table scaling.** Large `choose` lists degrade linearly and every non-matching handler still executes its predicate. Apps beyond a few dozen routes should use EndpointRouting; TechEmpower entries (where Giraffe has competed since Round 17, at times beating ASP.NET Core MVC in plaintext/JSON)[^5] reflect the optimized path, not a deep classic-router `choose` tree.
- **Major versions carry real breaks.** 5.0 split out ViewEngine and introduced EndpointRouting[^4]; 6.0 moved to .NET 6's native `task` computation expression; 7.0 targets modern .NET (8+). Upgrades are mechanical but not drop-in — expect namespace moves and serializer-default checks. Release notes in `RELEASE_NOTES.md` are the authoritative changelog.
- **Bus factor.** Maintenance is a three-person volunteer team after the founder stepped back. Issues get answered and PRs merge, but the contribution policy is openly conservative — features that add surface area are rejected by design. Plan on writing your own handlers for anything niche (the `HttpHandler` type makes this cheap).
- **Ecosystem gravity is C#.** Any ASP.NET Core library works, but samples will be in C# with `IServiceCollection` extension methods; budget translation effort. Conversely this is Giraffe's superpower: EF Core, SignalR, OpenTelemetry, and every auth provider just work.
- **51 open issues, single `master` release branch, pushed within the last week** (as of 2026-07): healthy for its size, but this is a slow-and-stable project, not a fast-moving one.

## When to Use / When Not

**Use when:**
- You write F# and want production web serving with Microsoft's runtime, hosting, and security machinery underneath.
- You value handler composition (`>=>`, `choose`) as the app-structuring primitive over MVC controllers or minimal-API lambdas.
- You need full ASP.NET Core ecosystem access (EF Core, auth providers, SignalR) without abandoning functional style.

**Avoid when:**
- Your team is C#-first: ASP.NET Core Minimal APIs cover the same ground with far more documentation and hiring pool.
- You want batteries included (scaffolding, ORM conventions, auth flows): Saturn layers that on top of Giraffe.
- You need cutting-edge performance-oriented F# routing: Oxpecker is a modern redesign by a former Giraffe contributor.
- You want a self-contained functional server with no ASP.NET Core dependency: that is Suave's design, not Giraffe's.

## Alternatives

- SuaveIO/suave — use instead when you want a standalone F# web server and ecosystem independence from ASP.NET Core; the project Giraffe was inspired by.
- SaturnFramework/Saturn — use instead when you want an opinionated MVC-style framework (built on Giraffe) with more conventions and less wiring.
- pimbrouwers/Falco — use instead when you want an even smaller functional F# toolkit over ASP.NET Core with a leaner API surface.
- Lanayx/Oxpecker — use instead when you want a Giraffe-style API rebuilt for modern .NET with endpoint routing as the only mode and performance as a headline goal.
- dotnet/aspnetcore — use Minimal APIs instead when the team is mixed C#/F# and documentation depth matters more than functional composition.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0-alpha | 2017-02 | Initial public releases under `dustinmoris/Giraffe`[^1]. |
| 1.0 | 2018-02 | First stable release[^6]. |
| — | 2018 | Repo moved to the `giraffe-fsharp` GitHub organization[^3]. |
| 3.x | 2018 | Continuation/Task-based core matured; performance-focused era[^2]. |
| 5.0 | 2021-01 | Giraffe.EndpointRouting; Giraffe.ViewEngine split into separate package[^4]. |
| 6.0 | 2022 | .NET 6 baseline; native F# `task` CE replaces external task builder. |
| 7.0 | 2024 | Modern .NET (8+) targeting; System.Text.Json-era defaults. |

## References

[^1]: Dustin Moris Gorski, "Functional ASP.NET Core" — the founding blog post. https://dusted.codes/functional-aspnet-core
[^2]: Gerard O'Connor, "Carry On! … Continuation over binding pipelines for functional web". https://medium.com/@gerardtoconnor/carry-on-continuation-over-binding-pipelines-for-functional-web-58bd7e6ea009
[^3]: Dustin Moris Gorski, "Evolving my open source project from a one man repository to an OSS organisation". https://dusted.codes/evolving-my-open-source-project-from-a-one-man-repository-to-a-proper-organisation
[^4]: Giraffe documentation (EndpointRouting, ViewEngine). https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md
[^5]: TechEmpower Framework Benchmarks — FSharp/giraffe implementation. https://github.com/TechEmpower/FrameworkBenchmarks/tree/master/frameworks/FSharp/giraffe
[^6]: Dustin Moris Gorski, "Announcing Giraffe 1.0.0" — 2018. https://dusted.codes/announcing-giraffe-100

## Tags

fsharp, dotnet, aspnet-core, web-framework, micro-framework, functional-programming, http-handler, middleware, server-side, view-engine
