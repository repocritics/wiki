# square/okhttp

> Square's HTTP client for the JVM, Android, and GraalVM — the de facto transport layer under most of the Android ecosystem.

[GitHub repo](https://github.com/square/okhttp) ·
[Official website](https://square.github.io/okhttp/) ·
[License: Apache-2.0](https://github.com/square/okhttp/blob/master/LICENSE.txt)

## Overview

OkHttp is an HTTP/HTTP2 client for the JVM and Android, first released by Square in 2013[^1]. It is one of the most widely deployed libraries in the JVM world: it is the default network transport under Retrofit, ships inside a large fraction of Android apps, and is the engine many higher-level SDKs (analytics, crash reporting, payment) embed without surfacing. With ~47k stars and ~9.3k forks, its influence is understated by GitHub metrics because most usage is transitive.

The library's defining stance is opinionatedness. It implements the modern HTTP RFCs (9110/9111/9112/9113) and, where a spec is ambiguous, mimics browser behavior rather than exposing a knob[^2]. It deliberately refuses configuration that exists only to work around broken servers or to emit invalid requests — it will not send a GET with a body, and its response cache is a concrete class rather than a pluggable interface. Teams that need to bend those rules are expected to reach for a different client. This is the central tradeoff: OkHttp gives you a correct, well-behaved user agent with connection pooling, transparent GZIP, and silent connection recovery, at the cost of being a poor fit when you need to do something the spec frowns on.

The codebase itself has been through two large migrations. The 3.x line was Java; 4.x (2019) was a source-compatible rewrite into Kotlin[^3]; 5.x (2024–) restructures the project as a Kotlin Multiplatform build, which changes how the artifact is consumed on Maven[^4]. Development remains active — commits land continuously against `master`.

## Getting Started

Gradle (Kotlin DSL), current stable line:

```kotlin
implementation("com.squareup.okhttp3:okhttp:5.4.0")
// optional but common:
implementation("com.squareup.okhttp3:logging-interceptor")
```

A synchronous GET, using try-with-resources so the response body is closed:

```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder().url(url).build();
  try (Response response = client.newCall(request).execute()) {
    return response.body().string();   // body is a one-shot stream
  }
}
```

A JSON POST:

```java
public static final MediaType JSON = MediaType.get("application/json");

String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(json, JSON);
  Request request = new Request.Builder().url(url).post(body).build();
  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

## Architecture / How It Works

The public API is small and immutable: `OkHttpClient`, `Request`, `Response`, `Call`. The client is a factory; a `Call` is a single request/response. Both sync (`execute()`) and async (`enqueue(callback)`) modes exist.

The core abstraction is the **interceptor chain**. Every call runs through an ordered list of interceptors, each of which can inspect, short-circuit, retry, or rewrite the request/response. There are two insertion points:

- **Application interceptors** — run once per `Call`, above the retry/redirect machinery. They see the logical request the app made and don't observe intermediate redirects.
- **Network interceptors** — run once per actual network request, below retries/redirects, so they see rewritten URLs, redirect hops, and served-from-cache short-circuits.

Below the user's interceptors sit OkHttp's built-in ones, in order: retry-and-follow-up, bridge (adds `Host`, `Content-Length`, cookies, transparent GZIP), cache, connect (obtains a connection), and the call-server interceptor that actually writes bytes. Understanding which layer you're in is the single most important mental model for using OkHttp correctly — the same interceptor placed at the two levels behaves very differently around caching and redirects.

Underneath, a **connection pool** keeps idle HTTP/1.1 and HTTP/2 connections alive for reuse (default: 5 idle connections, 5-minute keep-alive). HTTP/2 multiplexes all requests to one host over a single socket. A **Dispatcher** bounds concurrency for async calls (default 64 total, 5 per host). Routing includes automatic failover: if a host resolves to multiple IPs (IPv4+IPv6, redundant data centers), OkHttp tries alternates when the first connect fails. TLS is delegated to the platform provider, with optional Conscrypt/BoringSSL integration; TLS 1.3, ALPN, and certificate pinning are supported[^2].

I/O is not built on `java.io` — OkHttp depends on **Okio**[^5], Square's companion library, for its buffer and stream primitives. Okio is effectively a hard dependency, not an implementation detail you can swap.

## Production Notes

**Share one `OkHttpClient`.** The client owns the connection pool, thread pools, and dispatcher. Creating a new client per request defeats connection reuse and leaks threads. Derive variants with `client.newBuilder()` so they share the underlying pools.

**Bodies are one-shot and must be closed.** `Response.body()` is a live stream over a pooled connection. Failing to close it (or to fully read it) leaks the connection out of the pool; symptoms are exhausted connections and stalls under load, not immediate errors. Always use try-with-resources, and never call `body().string()` twice.

**Timeouts are off by default for reads.** Older defaults were effectively "wait forever." Set `connectTimeout`, `readTimeout`, `writeTimeout`, and `callTimeout` explicitly on the builder — the last bounds the entire call including redirects and retries, and is the one people forget.

**Android version floor moves.** OkHttp 5.x/4.x require Android 5.0+ (API 21) and Java 8+. The legacy `3.12.x` branch reaches back to API 9 / Java 7 but lacks TLS 1.2 and should be considered end-of-life for anything touching the public internet[^2]. Because OkHttp tracks the TLS ecosystem aggressively, staying current is a security posture, not just a feature choice.

**5.x Maven consumers must pick an artifact.** Now that the project is Kotlin Multiplatform, the bare `okhttp` artifact is empty in Maven — you must depend on `okhttp-jvm` or `okhttp-android` directly. Gradle resolves this automatically; Maven does not[^4]. The BOM (`okhttp-bom`) is the recommended way to keep the several artifacts version-aligned.

**Java 9 modules (5.2+) hide internals.** Packages under `okhttp3.internal.*` are no longer visible to module-based consumers. Code that reached into internals for platform tricks will fail to compile against a modular build; there is no supported replacement for most of it.

**MockWebServer is for basic tests, not a full server.** It ships in-repo for OkHttp's own testing and simple client tests, but is explicitly not developed as a general-purpose HTTP mocking tool — heavier needs (MockServer, WireMock) will outgrow it.

## When to Use / When Not

**Use when:**
- You're on the JVM or Android and want a correct, well-behaved HTTP client with connection pooling, HTTP/2, and TLS handled for you.
- You use Retrofit (which sits on OkHttp) or want to add interceptors for auth, logging, or retry.
- You value spec-correctness and sane defaults over maximum configurability.

**Avoid when:**
- You need to emit non-conforming requests (GET with body, pluggable cache implementations, arbitrary header manipulation the spec forbids) — OkHttp will fight you by design.
- You want a fully async, reactive, or Loom-first stack with no blocking-thread pool; OkHttp's model is threads + callbacks (coroutine support exists via `okhttp3.coroutines` but the core is thread-based).
- You're on a non-JVM platform — despite the KMP restructure, this is a JVM/Android/GraalVM library, not a browser or native-first client.

## Alternatives

- square/retrofit — not a replacement but the typed REST layer you usually want *on top of* OkHttp.
- ktor (ktorio/ktor) — Kotlin-first, coroutine-native client with pluggable engines; use when you want suspend-based APIs and multiplatform reach beyond the JVM.
- OpenFeign/feign — declarative JVM HTTP client; use when you want annotation-driven interfaces over an existing HTTP backend.
- apache/httpcomponents-client — the older, highly configurable JVM client; use when you specifically need the customization OkHttp refuses to expose.
- The JDK `java.net.http.HttpClient` (Java 11+) — use when you want zero dependencies and HTTP/2 from the standard library and can accept a thinner feature set.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013 | Initial release by Square[^1]. |
| 2.0 | 2014 | Reworked API; `OkHttpClient`/`Call` shape. |
| 3.0 | 2016 | New immutable request/response API, interceptor model. |
| 3.12.x | 2018 | Last branch supporting Android API 9 / Java 7 (no TLS 1.2). |
| 4.0 | 2019 | Source-compatible rewrite to Kotlin[^3]. |
| 5.0 | 2024 | Kotlin Multiplatform restructure; split JVM/Android artifacts[^4]. |
| 5.4.0 | 2026 | Current stable release line[^6]. |

## References

[^1]: OkHttp changelog / release history. https://square.github.io/okhttp/changelogs/changelog/
[^2]: OkHttp README, "A well behaved user agent" and Requirements sections. https://github.com/square/okhttp/blob/master/README.md
[^3]: "OkHttp 4" — Square Developer blog / release notes on the Kotlin rewrite. https://square.github.io/okhttp/changelogs/changelog_4x/
[^4]: OkHttp README, "Maven and JVM Projects" — Kotlin Multiplatform artifact split. https://github.com/square/okhttp/blob/master/README.md
[^5]: Okio — Square's I/O library and OkHttp's buffer/stream dependency. https://github.com/square/okio
[^6]: Maven Central — com.squareup.okhttp3:okhttp:5.4.0. https://central.sonatype.com/artifact/com.squareup.okhttp3/okhttp

## Tags

http-client, kotlin, java, android, jvm, networking, http2, tls, retrofit, graalvm, library
