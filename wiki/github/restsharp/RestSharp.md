# restsharp/RestSharp

> A convenience wrapper over .NET's `HttpClient` — parameter handling, serialization, and auth, not a transport of its own.

[GitHub repo](https://github.com/restsharp/RestSharp) ·
[Official website](https://restsharp.dev) ·
[License: Apache-2.0](https://github.com/restsharp/RestSharp/blob/dev/LICENSE.txt)

## Overview

RestSharp is one of the oldest REST/HTTP client libraries in the .NET ecosystem, created by John Sheehan around 2009[^1] and now maintained under the .NET Foundation. It is not an HTTP stack — since version 107 it is a thin layer over `System.Net.Http.HttpClient`[^2]. What it adds on top is ergonomics: typed request objects, uniform handling of query/URL-segment/header/cookie/body parameters, pluggable JSON/XML/CSV serialization, and a set of authenticators.

The defining tension in RestSharp's history is its 2021 rewrite. For its first decade the library was built on the legacy `HttpWebRequest` transport, exposed large `IRestClient` / `IRestRequest` / `IHttp` interfaces, and shipped a bundled `SimpleJson` serializer. Version 107 tore most of that out: it moved to `HttpClient`, made the client configuration immutable via `RestClientOptions`, deprecated the fat interfaces, and swapped `SimpleJson` for `System.Text.Json`[^2]. The rewrite fixed long-standing thread-safety and socket-exhaustion problems but broke essentially every non-trivial codebase that upgraded across it.

The second thing to know is the support model. The README states plainly that RestSharp is an open-source project with a single maintainer and that issues are unlikely to be addressed unless they affect a large group of users; the recommended path is to fork and fix it yourself[^3]. Treat it as stable, mature, low-velocity infrastructure rather than an actively expanding framework.

## Getting Started

```bash
dotnet add package RestSharp
```

```csharp
using RestSharp;

// RestClient wraps an HttpClient — create ONE and reuse it.
var options = new RestClientOptions("https://api.example.com");
var client = new RestClient(options);

var request = new RestRequest("posts/{id}")
    .AddUrlSegment("id", 42)
    .AddQueryParameter("fields", "title");

// GetAsync<T> executes and deserializes in one call.
var post = await client.GetAsync<Post>(request);

record Post(int Id, string Title);
```

POST with a JSON body:

```csharp
var create = new RestRequest("posts", Method.Post)
    .AddJsonBody(new { title = "hello" });

var response = await client.ExecutePostAsync(create);
```

## Architecture / How It Works

At the center is `RestClient`, which owns (or is handed) an `HttpClient` instance. Configuration that used to live on the client — base URL, default parameters, timeout, redirect policy, TLS options — now lives on the immutable `RestClientOptions` passed at construction. This immutability is deliberate: it makes the client safe to share across threads, which the pre-107 mutable client was not.

A `RestRequest` is a declarative bag of parameters. Every input — query string, URL segment (`{placeholder}` substitution), header, cookie, and body — is a `Parameter` with a `ParameterType`, and RestSharp assembles them into an `HttpRequestMessage` at execution time. Bodies are produced by a serializer: the core package ships `System.Text.Json` plus a basic XML serializer, and separate packages provide `Newtonsoft.Json`, a richer XML serializer, and `CsvHelper` for CSV[^3]. Serializers are registered per-client via `UseSerializer`, so you can mix request and response formats.

Execution flows through `ExecuteAsync`, which returns a `RestResponse` carrying status, headers, raw content, and (for the generic overloads) a deserialized `.Data`. The many convenience methods — `GetAsync<T>`, `ExecutePostAsync`, `PostAsync<T>` — are extension methods over that core. Authentication is an `IAuthenticator` on the client (`HttpBasicAuthenticator`, `JwtAuthenticator`, OAuth1 via a separate package) that mutates the outgoing request. Because everything ultimately becomes an `HttpRequestMessage`, you can also inject a custom `HttpMessageHandler` (for Polly resilience policies, mock handlers in tests, or a shared `SocketsHttpHandler`) through the options.

## Production Notes

**Reuse the client — do not `new` one per request.** Because `RestClient` wraps `HttpClient`, it inherits `HttpClient`'s most infamous footgun: constructing and disposing one per call exhausts the OS socket pool under load (sockets linger in `TIME_WAIT`). Register `RestClient` as a singleton, or build it from a pooled `HttpClient`/`IHttpClientFactory` and pass it in. This was a genuine failure mode in the pre-107 design and is still a failure mode if you misuse the new API.

**The 107 upgrade is a rewrite, not a version bump.** Migrating across it means: replacing `IRestClient`/`IRestResponse` references with concrete types, moving client config into `RestClientOptions`, rewriting synchronous calls, and re-checking JSON behavior after the `SimpleJson` → `System.Text.Json` switch — `System.Text.Json` is stricter and case-sensitive by default, so payloads that round-tripped under the old serializer can silently deserialize to nulls[^2]. Teams frequently pin to the last 106.x release rather than pay this cost.

**Async is the supported path.** 107 removed the first-class synchronous execution API; synchronous usage now means blocking on the async methods, which reintroduces deadlock risk in legacy `SynchronizationContext` environments (classic ASP.NET, WPF/WinForms UI threads). New code should be async end to end.

**Timeouts and cancellation.** `RestClientOptions.MaxTimeout` (milliseconds) sets a per-request timeout; a value of 0 means no RestSharp-level timeout and you fall back to `HttpClient`'s default. Always pass a `CancellationToken` to `ExecuteAsync` for real cancellation rather than relying on timeout alone.

**Error handling is opt-in.** By default RestSharp does not throw on non-2xx responses; it returns a `RestResponse` with `IsSuccessful == false` and populates `ErrorException` / `ErrorMessage` for transport-level failures. Set `ThrowOnAnyError` (or `ThrowOnDeserializationError`) on the options if you prefer exceptions. Code that assumes an HTTP 500 throws will otherwise sail past the failure.

**Support expectations.** With a single maintainer and an explicit "fix it yourself" policy, do not build a delivery plan around upstream bug fixes. The dependency is small and vendorable — that is part of why it remains viable.

## When to Use / When Not

**Use when:**
- You want a fluent, parameter-oriented wrapper over `HttpClient` without hand-assembling `HttpRequestMessage` objects.
- You consume messy third-party APIs and want uniform handling of query/segment/header/body params and swappable serializers (JSON, XML, CSV, Newtonsoft).
- You have an existing RestSharp codebase — it is stable and worth keeping.
- You want built-in Basic/JWT/OAuth1 authenticators without pulling a larger framework.

**Avoid when:**
- You control both ends and speak plain JSON — `HttpClient` + `System.Net.Http.Json` extensions cover that with zero dependencies.
- You want compile-time-generated, interface-driven clients — Refit or Kiota fit better.
- You need a heavily supported library with a responsive maintainer and frequent releases.
- Your API is described by OpenAPI and you would rather generate the client than write requests by hand.

## Alternatives

- reactiveui/refit — declarative, interface-based REST client; you define a C# interface and it generates the `HttpClient` calls. Use instead when you want type-safe, boilerplate-free clients for a well-defined API.
- tmenier/Flurl — fluent URL builder plus `HttpClient` wrapper with a strong testing story. Use instead when you want inline fluent requests and first-class test fakes over a separate request object model.
- microsoft/kiota — generates full API clients from an OpenAPI description. Use instead when the service ships a spec and you want the client generated, not authored.
- dotnet/runtime (`HttpClient` + `System.Net.Http.Json`) — the built-in baseline. Use instead when your needs are simple JSON calls and you want no third-party dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0-ish | 2009 | Created by John Sheehan; built on `HttpWebRequest`[^1]. |
| 100–106 | 2010s | Long-lived line: `IRestClient`/`IHttp`, `SimpleJson`, mutable client. |
| 107.0 | 2021-11 | Rewrite onto `HttpClient`; `RestClientOptions`; `System.Text.Json`; interfaces deprecated[^2]. |
| 108.0 | 2022 | Follow-up fixes and refinements after the 107 break. |
| 110+ | 2023–2024 | Modern `net`/`netstandard` targets, incremental releases. |
| 112.x | 2024–2025 | Current-line maintenance under .NET Foundation. |

Version dates below 107 are approximate; consult the NuGet release history for exact numbers[^4].

## References

[^1]: RestSharp origin and .NET Foundation membership. https://dotnetfoundation.org/projects
[^2]: RestSharp v107 migration notes — move to `HttpClient`, `RestClientOptions`, `System.Text.Json`, deprecated interfaces. https://restsharp.dev/docs/intro/v107/
[^3]: RestSharp README — package layout and single-maintainer support policy. https://github.com/restsharp/RestSharp/blob/dev/README.md
[^4]: RestSharp on NuGet — full version and release history. https://www.nuget.org/packages/RestSharp

## Tags

csharp, dotnet, http-client, rest-client, api-client, serialization, httpclient-wrapper, networking, nuget-package, dotnet-foundation
