# reactiveui/refit

> Type-safe REST client for .NET: you declare a C# interface, Refit generates the `HttpClient` plumbing that implements it.

[GitHub repo](https://github.com/reactiveui/refit) ·
[Official website](https://reactiveui.github.io/refit/) ·
[License: MIT](https://github.com/reactiveui/refit/blob/main/LICENSE)

## Overview

Refit turns a REST API into a plain C# interface. You annotate methods with
HTTP attributes (`[Get("/users/{id}")]`), and Refit produces an implementation
that builds the `HttpRequestMessage`, serializes the body, appends the query
string, deserializes the response, and hands you back a `Task<T>`. It is a
direct port of the idea behind Square's Retrofit[^1] from the JVM to .NET, and
it occupies the same niche: the declarative middle ground between hand-rolling
`HttpClient` calls and generating a full client from an OpenAPI spec.

The project was started by Paul Betts and is now maintained under the ReactiveUI
organization[^2]. It is one of the more widely used HTTP client libraries in the
.NET ecosystem — roughly 9,500 stars and ~790 forks as of mid-2026, on a
codebase that has existed since 2013 and remains actively developed (last push
July 2026). Its reach is broad: the library targets modern .NET (8/9/10/11),
.NET Framework 4.6.2+, Blazor, WinUI, and the Uno Platform.

The defining tension in current Refit is reflection versus source generation.
For most of its history Refit built requests at runtime via reflection. Since
the move to Roslyn source generators, the default path emits request-building
code at compile time, which is what makes Refit usable under trimming and Native
AOT — but the reflection fallback still exists for interface shapes the
generator cannot yet emit, and understanding which path a given method takes is
the key to using Refit well on constrained runtimes.

## Getting Started

```bash
dotnet add package Refit
# optional: DI integration and Newtonsoft serializer
dotnet add package Refit.HttpClientFactory
dotnet add package Refit.Newtonsoft.Json
```

```csharp
using Refit;

public interface IGitHubApi
{
    [Get("/users/{user}")]
    Task<User> GetUser(string user);
}

// Standalone:
var gitHubApi = RestService.For<IGitHubApi>("https://api.github.com");
var octocat = await gitHubApi.GetUser("octocat");

// Or via HttpClientFactory (recommended for apps with DI):
services
    .AddRefitClient<IGitHubApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri("https://api.github.com"));
```

Any method parameter not consumed by a `{placeholder}` in the route becomes a
query-string parameter automatically — a deliberate divergence from Retrofit,
where every parameter must be annotated.

## Architecture / How It Works

The core of Refit is the `RestService.For<T>()` factory and a set of Roslyn
source generators shipped inside the `Refit` package (no separate analyzer
package needed). At build time the generator emits a concrete class per
interface. By default it also emits the request-building code inline: the
generated method constructs the `HttpRequestMessage` directly — path
substitution, query parameters, headers, `[Body]`, `[Multipart]` parts — and
dispatches through Refit's runtime helpers, avoiding runtime reflection,
metadata lookup, and argument boxing.

On .NET 5+ the generator also emits a module initializer that registers each
client factory at assembly load, so `RestService.ForGenerated<T>` and
`AddRefitGeneratedClient<T>` resolve clients with zero reflection — the path
required for trimmed and AOT builds. On .NET Framework there is no
`ModuleInitializerAttribute` in the BCL, so `RestService.For<T>` falls back to a
runtime type lookup and the reflection request builder, which is why .NET
Framework consumers must reference the opt-in `Refit.Reflection` package.

Not every interface shape can be generated. An unconstrained generic parameter
(`Task<T> Get<T>(T request)`) resolves its bound properties only at call time, so
those methods fall back to the reflection builder, as do object query parameters
serialized through the content serializer and some exotic multipart shapes. The
generator emits inline code for the common cases and defers the rest — behavior
you can force off with `<RefitGeneratedRequestBuilding>false</...>` or disable
entirely with `<DisableRefitSourceGenerator>true</...>`.

Serialization is pluggable via `RefitSettings.ContentSerializer` (default
`System.Text.Json`; Newtonsoft and XML available as separate packages).
Cross-cutting concerns — auth, retries, logging — are handled the idiomatic
`HttpClient` way through `DelegatingHandler` chains and Polly, not Refit-specific
hooks.

## Production Notes

- **Reflection vs. AOT is the main footgun.** If you build for Native AOT or
  aggressive trimming and use `RestService.For<T>`/`AddRefitClient<T>` with an
  interface that falls back to reflection (unconstrained generics, object query
  params), it can break at runtime after passing at compile time. Use
  `ForGenerated<T>`/`AddRefitGeneratedClient<T>`, which never fall back and throw
  loudly if no generated implementation exists, to keep the reflection path out
  entirely.
- **`.NET Framework` needs `Refit.Reflection`.** Because module initializers are
  a .NET 5+ feature, `ForGenerated`/`AddRefitGeneratedClient` throw on .NET
  Framework unless you register factories yourself. Raising `<LangVersion>` does
  not help — the missing type is the blocker, not the language version.
- **URL resolution has two modes with different semantics.** The legacy default
  requires relative routes to start with `/`, prepends the base-address path, and
  trims a trailing slash. `UrlResolutionMode.Rfc3986` matches `HttpClient`/`Uri`
  behavior, where a leading `/` replaces the base path and the base's trailing
  slash is significant. Teams migrating base URLs get surprised here.
- **Validation timing moved with generation.** Under generated request building a
  malformed route (e.g., missing leading slash under legacy resolution) surfaces
  its `ArgumentException` on the first call, not from `RestService.For<T>(...)` —
  fail-fast startup checks that relied on construction throwing need adjusting.
- **Error handling is opt-in by return type.** `Task<T>` throws `ApiException` on
  non-success status; `Task<IApiResponse<T>>` / `Task<ApiResponse<T>>` return the
  response without throwing so you inspect `IsSuccessStatusCode` yourself. Mixing
  the conventions inconsistently is a common source of unhandled exceptions.
- **Compile-time analyzers catch real mistakes.** Refit flags backslash routes,
  multiple `[Body]` or `CancellationToken` parameters, and wrong
  `[HeaderCollection]` types, with code fixes for the mechanical ones — don't
  suppress them blindly, they map to runtime failures.

## When to Use / When Not

**Use when:**
- You call one or more REST APIs from .NET and want a typed contract instead of
  scattered `HttpClient` string-building.
- You want DI-native clients (`AddRefitClient`) with `HttpClientFactory`,
  handlers, and Polly resilience.
- You target trimming/Native AOT and can stay on the generated path.

**Avoid when:**
- The API is not cleanly RESTful (heavy custom protocols, streaming RPC, GraphQL)
  — a purpose-built client or raw `HttpClient` fits better.
- You want a full client generated from an OpenAPI/Swagger document with no
  hand-written interface — Kiota or NSwag is the better shape.
- You need per-call imperative control over every request; a fluent client like
  Flurl or RestSharp is less indirection than fighting attribute conventions.

## Alternatives

- canton7/RestEase — the closest direct competitor; same interface-attribute
  model. Use it if you prefer its API surface, though Refit has broader adoption
  and AOT investment.
- microsoft/kiota — generates a full client from an OpenAPI description. Use when
  you have a spec and want the whole client, not a hand-written interface.
- restsharp/RestSharp — imperative, request-builder style client. Use when you
  want explicit per-call control rather than declarative interfaces.
- tmenier/Flurl — fluent URL builder plus HTTP. Use for ad-hoc calls and testable
  fluent chains without defining interfaces.
- System.Net.Http (`HttpClient` + `System.Net.Http.Json`) — the built-in
  baseline. Use when a single call or two doesn't justify a dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2013 | Created by Paul Betts; a .NET port of Square's Retrofit[^1][^2]. |
| 6.x | ~2021 | Roslyn source generators; `System.Text.Json` as default serializer, Newtonsoft split into `Refit.Newtonsoft.Json`[^3]. |
| 7.x–8.x | 2022–2023 | HttpClientFactory maturation, DI ergonomics, expanded generated request coverage. |
| 14.x | 2025–2026 | Reflection request builder moved into the opt-in `Refit.Reflection` package; generated path is default for trimming/Native AOT[^3]. |

## References

[^1]: Square's Retrofit — the JVM library Refit is modeled on. https://square.github.io/retrofit/
[^2]: Refit repository and documentation, ReactiveUI organization. https://github.com/reactiveui/refit
[^3]: Refit "Breaking changes and release notes." https://github.com/reactiveui/refit/blob/main/docs/breaking-changes.md

## Tags

csharp, dotnet, rest-client, http, source-generator, api-client, type-safe, retrofit-inspired, native-aot, httpclient
