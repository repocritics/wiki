# square/retrofit

> A type-safe HTTP client for Android and the JVM that turns an annotated Java/Kotlin interface into a working API client.

[GitHub repo](https://github.com/square/retrofit) ·
[Official website](https://square.github.io/retrofit/) ·
[License: Apache-2.0](https://github.com/square/retrofit/blob/trunk/LICENSE.txt)

## Overview

Retrofit is Square's declarative HTTP client for the JVM. You describe a remote
API as a Java or Kotlin interface — methods annotated with `@GET`, `@POST`,
`@Path`, `@Query`, `@Body` and friends — and Retrofit generates a concrete
implementation at runtime via a dynamic proxy. It does not do networking
itself: it is a thin type-safe layer over OkHttp (also by Square), which owns
connection pooling, HTTP/2, TLS, timeouts, and interceptors[^1]. Serialization
is delegated to pluggable converters, the return type to pluggable call
adapters; Retrofit's own job is the mapping between an annotated method
signature and an OkHttp `Request`/`Response`.

It has been the default HTTP layer on Android for roughly a decade and is
equally usable server-to-server on plain JVM. The current line is 3.x
(`com.squareup.retrofit2:retrofit:3.0.0`), which raised the floor to Java 8+ or
Android API 21+[^2]. The library is mature and low-churn — stable since
Retrofit 2 and treated as essentially feature-complete rather than fast-moving.

The defining tradeoff: Retrofit is minimal on purpose. It gives you a clean,
compile-time-checked API surface, but every operational concern (retries,
caching, auth refresh, logging, timeouts) lives one layer down in OkHttp — you
get little unless you configure the `OkHttpClient` you hand it. Teams expecting
a batteries-included framework are surprised by how much is delegated.

## Getting Started

Gradle:

```kotlin
dependencies {
    implementation("com.squareup.retrofit2:retrofit:3.0.0")
    implementation("com.squareup.retrofit2:converter-moshi:3.0.0")
}
```

Define the API as an interface and build a client. Kotlin `suspend` functions
are supported directly:

```kotlin
data class Repo(val id: Long, val name: String)

interface GitHubService {
    @GET("users/{user}/repos")
    suspend fun listRepos(@Path("user") user: String): List<Repo>
}

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.github.com/")   // must end in '/'
    .addConverterFactory(MoshiConverterFactory.create())
    .build()

val service = retrofit.create(GitHubService::class.java)
val repos = service.listRepos("square")   // runs on OkHttp's dispatcher
```

Returning `Response<List<Repo>>` instead of `List<Repo>` gives access to
status code and headers without an exception on non-2xx responses.

## Architecture / How It Works

`Retrofit.create(Service.class)` returns a `java.lang.reflect.Proxy`. On the
first call to a method, Retrofit parses its annotations into a `ServiceMethod`
(an HTTP verb, a relative URL template, and parameter handlers), caches it, and
reuses the parsed form thereafter. `validateEagerly(true)` forces this parsing
at `create()` time so malformed interfaces fail fast instead of on first call.

Three extension points define everything:

- **Converters** (`Converter.Factory`) turn bodies into types. Official ones
  include Moshi, Gson, Jackson, kotlinx.serialization, Protobuf, Wire, and
  Scalars (plain `String`/primitives)[^3]. Factories are tried in registration
  order; the first that claims a type wins. A missing or wrong converter is the
  most common source of cryptic "could not locate ResponseBody converter" errors.
- **Call adapters** (`CallAdapter.Factory`) map the built-in `Call<T>` to other
  return types: RxJava `Observable`/`Single`, Guava/Java8 `CompletableFuture`,
  or — since 2.6.0 — Kotlin `suspend` functions via a built-in adapter[^4].
- **`Call<T>`** is the unit of work: one request, executed synchronously
  (`execute()`) or asynchronously (`enqueue()`), one-shot but `clone()`-able.

The coupling to OkHttp is deliberate and total. Retrofit builds an OkHttp
`Request`, hands it to the `OkHttpClient` you provided (or a default one), and
wraps the `Response`. Interceptors, the connection pool, the dispatcher thread
pool, DNS, timeouts, and the cache are all OkHttp concerns; Retrofit adds no
threading model of its own beyond a platform-default callback executor (on
Android, callbacks are marshalled to the main thread).

## Production Notes

**Non-2xx does not throw for `Call`/`Response`.** `response.isSuccessful()`
must be checked explicitly; a 404 or 500 is a perfectly "successful" call at the
transport level. This trips up almost everyone once. With `suspend` functions
the behavior forks: declaring the return as the body type `T` throws
`HttpException` on non-2xx, while declaring `Response<T>` never throws for HTTP
status and returns the error response for inspection.

**Everything operational lives in OkHttp.** Timeouts, retry-on-failure,
response caching, logging (`HttpLoggingInterceptor`), and auth-token refresh
(`Authenticator`) are configured on the `OkHttpClient`, not Retrofit. Share a
single `OkHttpClient` across Retrofit instances via `newBuilder()` so they reuse
the connection pool and thread pool — separate clients silently multiply socket
and thread usage.

**R8 / ProGuard.** Because interface methods are read reflectively, shrinking
can strip generic signatures. R8 ships the required keep rules automatically;
plain ProGuard users must add the rules from `retrofit2.pro` (and typically the
OkHttp rules)[^2]. Missing rules surface as runtime failures parsing generic
return types, not build errors.

**Converters and empty bodies.** Returning a raw `String` requires the Scalars
converter, or Retrofit tries to JSON-decode it. An HTTP 204/205 empty body must
map to a nullable/`Unit` type or deserialization fails on the empty stream. For
tests, the `retrofit-mock` module (`MockRetrofit` + `BehaviorDelegate`) stubs
the interface with simulated latency and failure rates.

**Upgrade history matters.** Retrofit 1 → 2 was a ground-up rewrite: the package
changed from `retrofit` to `retrofit2`, the `Call` model replaced synchronous
interface methods, and converters/adapters were extracted into separate
artifacts — the two majors do not mix. Retrofit 2 → 3 is a far smaller step
whose headline change is the raised Java 8 / API 21 baseline[^2]; most 2.x code
compiles unchanged.

## When to Use / When Not

**Use when:**
- You're on Android or the JVM and want a compile-time-checked, annotation-based
  REST client with minimal boilerplate.
- You already use (or are happy to use) OkHttp and want its interceptor/caching
  ecosystem underneath.
- You want a stable, well-understood dependency that rarely breaks across
  upgrades.

**Avoid when:**
- You target Kotlin Multiplatform (iOS, JS, native) — Retrofit is JVM-only; Ktor
  is the multiplatform answer.
- Your API is GraphQL — use a GraphQL client (Apollo) rather than modeling
  every operation as a REST method.
- You want a batteries-included framework with built-in retry/circuit-breaking
  policy — that lives in OkHttp or a higher-level stack, not Retrofit.

## Alternatives

- ktorio/ktor — Kotlin-first, coroutine-native, multiplatform HTTP client; use instead when you need KMP targets or prefer a DSL over annotations.
- square/okhttp — the engine Retrofit sits on; use directly when you want raw request/response control without the typed interface.
- OpenFeign/feign — declarative annotation-based HTTP client for server-side JVM/Spring services; use instead in a backend microservice mesh.
- apollographql/apollo-kotlin — use instead when the API is GraphQL rather than REST.
- Volley — legacy Android networking; only relevant when maintaining old apps, not for new work.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013 | Square's declarative HTTP client; synchronous interface methods, bundled JSON. |
| 2.0 | 2016-03 | Full rewrite on OkHttp/Okio; `Call` model, extracted converters and call adapters[^1]. |
| 2.6.0 | 2019-06 | Built-in Kotlin `suspend` function support[^4]. |
| 3.0.0 | 2025 | Baseline raised to Java 8+ / Android API 21+; current line[^2]. |

## References

[^1]: Retrofit website — overview and design. https://square.github.io/retrofit/
[^2]: Retrofit README — coordinates `com.squareup.retrofit2:retrofit:3.0.0`, Java 8+ / Android API 21+ minimum, R8/ProGuard rules (`retrofit2.pro`). https://github.com/square/retrofit
[^3]: Retrofit converter modules (Moshi, Gson, Jackson, kotlinx.serialization, Protobuf, Wire, Scalars). https://github.com/square/retrofit/tree/trunk
[^4]: Retrofit CHANGELOG — 2.6.0 added Kotlin `suspend` function support. https://github.com/square/retrofit/blob/trunk/CHANGELOG.md

## Tags

java, kotlin, android, http-client, rest, networking, jvm, okhttp, api-client, type-safe
