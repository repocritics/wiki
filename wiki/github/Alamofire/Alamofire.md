# Alamofire/Alamofire

> An HTTP networking library for Swift, built as a convenience layer over Foundation's URLSession.

[GitHub repo](https://github.com/Alamofire/Alamofire) ·
[Documentation](https://alamofire.github.io/Alamofire/) ·
[License: MIT](https://github.com/Alamofire/Alamofire/blob/master/LICENSE)

## Overview

Alamofire is the de facto standard HTTP client for Apple-platform Swift development. It began in 2014 as the Swift successor to AFNetworking, the Objective-C library by the same original author (Mattt Thompson); the "AF" prefix and the `AF` namespace carry over from that lineage[^1]. It is now owned and maintained by the Alamofire Software Foundation (ASF), with Jon Shier as the primary long-running maintainer.

The library does not implement its own networking stack. Every request ultimately runs through `URLSession`, and Alamofire's value is the ergonomics layered on top: chainable request/response builders, response validation, decodable serialization on a background queue, multipart form encoding, request retry and adaptation via interceptors, TLS certificate/public-key pinning, network reachability, and cURL command output for debugging. This framing matters because it defines the project's central tension: since Apple shipped `async/await` on `URLSession` (iOS 15), the baseline "make a request, decode JSON" case no longer needs a dependency. Alamofire's continued relevance rests on the pieces `URLSession` still does not give you cheaply — interceptor-based retry, `ServerTrustManager` pinning, multipart uploads, event monitors, and a uniform response-serialization pipeline.

As of 2026 the current major line is 5.x (5.11.x), a 2020 rewrite that remains source-stable[^2]. The library is actively maintained but has settled into a mature, low-churn cadence — most activity is Swift-version compatibility, concurrency (`Sendable`) hardening, and incremental additions like `WebSocketRequest`.

## Getting Started

Swift Package Manager (add to `Package.swift`):

```swift
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire.git", .upToNextMajor(from: "5.11.0"))
]
```

CocoaPods: `pod 'Alamofire'`. Carthage: `github "Alamofire/Alamofire"`.

```swift
import Alamofire

struct User: Decodable { let id: Int; let name: String }

// AF is the shared default Session. async/await, validation, background decode.
let user = try await AF.request("https://api.example.com/users/1")
    .validate()                       // 200..<300 + acceptable Content-Type
    .serializingDecodable(User.self)
    .value
```

## Architecture / How It Works

The central object is `Session`, which wraps a `URLSession` plus its delegate and a set of serial `DispatchQueue`s. The global `AF` is just a default `Session` singleton. In 5.0 this replaced the older `SessionManager` type[^2]. A `Session` owns three roles: creating requests, driving the underlying `URLSession` delegate callbacks, and dispatching response serialization off the network queue.

Requests are modeled as a class hierarchy — `DataRequest`, `DownloadRequest`, `UploadRequest`, `DataStreamRequest`, `WebSocketRequest` — all descending from `Request`. A `Request` is stateful and retained by its `Session` until it finishes; this is the single most important thing to internalize (see Production Notes). Response handling is a chain: `.validate()` attaches a validator, `.serializingDecodable(_:)` / `.responseData` attach serializers, and the terminal `.response` / `.value` awaits completion. Serialization runs on a dedicated queue, not the main thread, then hops back.

Cross-cutting behavior is injected through protocols rather than subclassing:

- **`RequestInterceptor`** = `RequestAdapter` (mutate outgoing `URLRequest`, e.g. attach an auth token) + `RequestRetrier` (decide whether to retry on failure). `.retryPolicy` is a built-in interceptor. This is the mechanism for OAuth refresh-and-retry flows.
- **`ServerTrustManager` / `ServerTrustEvaluating`** — certificate and public-key pinning, evaluated in the TLS challenge callback.
- **`ParameterEncoder`** (`URLEncodedFormParameterEncoder`, `JSONParameterEncoder`) — replaced the older `ParameterEncoding` protocol for `Encodable` parameters.
- **`EventMonitor`** — observation hooks for logging, metrics, and integration with tools; how you'd wire request timing into your own telemetry.

`Result`-typed responses (`AFDataResponse<T>` carrying `.success`/`.failure` with `AFError`) and Combine `DataResponsePublisher` support are also part of the 5.x surface. Swift Concurrency (`async/await`) support is layered on top of the same serializer chain.

## Production Notes

**Keep your `Session` alive.** A `Request` is retained by the `Session` that created it; if the `Session` deallocates, in-flight requests are cancelled. The classic bug is constructing a `Session` as a local variable or spinning up a fresh `Session` per request — the request silently dies or you leak sockets. Create one `Session` (or use `AF`) and hold it for the app/service lifetime.

**Certificate pinning is a rotation footgun.** `ServerTrustManager` pinning to leaf certificates breaks the moment the server rotates its cert. Pin to public keys or to an intermediate CA, and always ship a plan for rotation. In `debug` builds `ServerTrustManager` defaults to enforcing evaluation for all evaluated hosts unless configured otherwise — misconfiguration surfaces as hard TLS failures, not warnings.

**Interceptor retry loops.** `RequestRetrier` gives you retry, but a retrier that always returns `.retry` on a persistent 401/500 will loop until it exhausts your policy — bound retries and don't retry non-idempotent requests blindly.

**Linux/Windows/Android are unsupported.** The library *builds* on these via `swift-corelibs-foundation`, but certificate pinning (`ServerTrustManager`), some HTTP auth challenges (can crash), `CachedResponseHandler`, `URLSessionTaskMetrics`, and `WebSocketRequest` are unavailable or broken[^3]. Treat Alamofire as an Apple-platform library; for server-side Swift use a native stack instead.

**Do you even need it?** For a codebase that only does JSON GET/POST on iOS 15+, native `URLSession` `async/await` covers it with zero dependency and smaller binary. Reach for Alamofire when you want interceptor-based auth-refresh-and-retry, multipart uploads, pinning, or a uniform serialization/validation pipeline across many endpoints — not for one or two calls.

**Swift 6 concurrency.** Recent releases target Swift 6.0–6.2 / Xcode 16 and have been hardened for strict concurrency (`Sendable`)[^4]. Adopting strict concurrency in your own app can surface warnings at Alamofire boundaries (closures, custom serializers) — expect some annotation work.

## When to Use / When Not

**Use when:**
- You need OAuth token refresh with automatic retry (`RequestInterceptor`).
- You do multipart uploads, download-to-file with resume data, or streamed responses.
- You want certificate/public-key pinning without hand-rolling TLS challenge handling.
- You want one consistent validation + `Decodable` serialization pipeline across a large API surface, plus cURL/event-monitor debugging.

**Avoid when:**
- Your app makes a handful of plain requests on iOS 15+ — native `URLSession` `async/await` is enough.
- You're on Linux/Windows/Android or doing server-side Swift — it's unsupported there.
- You want to minimize binary size and dependency surface.
- You need a high-level, target-typed abstraction — layer Moya on top rather than using Alamofire directly.

## Alternatives

- apple/swift-nio — event-driven, non-blocking network I/O for server-side Swift; use when you need a low-level cross-platform stack, not an Apple-client convenience layer.
- Moya/Moya — enum-based API abstraction built on top of Alamofire; use when you want endpoints modeled as typed targets rather than raw request builders.
- kean/Get — small async/await web-API client; use when you want modern concurrency ergonomics with a much smaller footprint than Alamofire.
- (native) URLSession — use when iOS 15+ `async/await` covers your needs and you want zero dependencies.
- Alamofire/AlamofireImage — companion library for image serialization/caching; use alongside Alamofire when you need image download + in-memory cache.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-09 | Initial release; Swift successor to AFNetworking[^1]. |
| 3.0 | 2015-10 | Error-handling redesign, Swift 2.x. |
| 4.0 | 2016-09 | Swift 3 migration, API modernization. |
| 5.0 | 2020-04 | Major rewrite: `Session`, `Request` subclasses, `RequestInterceptor`, `Result`/`AFError`, `ParameterEncoder`, `ServerTrustManager`[^2]. |
| 5.x | 2021–2024 | Swift Concurrency + Combine support; `WebSocketRequest`; `Sendable`/Swift 6 hardening[^4]. |
| 5.11.x | 2025–2026 | Current line; Swift 6.0–6.2 / Xcode 16, ongoing compatibility maintenance. |

## References

[^1]: Alamofire is named after the Alamo Fire flower and originated as the Swift successor to AFNetworking. README, "FAQ" and Credits. https://github.com/Alamofire/Alamofire#faq
[^2]: Alamofire 5.0 Migration Guide (introduces `Session`, `Request` subclasses, `RequestInterceptor`, `ParameterEncoder`, `ServerTrustManager`). https://github.com/Alamofire/Alamofire/blob/master/Documentation/Alamofire%205.0%20Migration%20Guide.md
[^3]: README, "Known Issues on Linux and Windows" — unsupported platform limitations. https://github.com/Alamofire/Alamofire#known-issues-on-linux-and-windows
[^4]: README, "Requirements" — Swift 6.0/6.1/6.2, Xcode 16.0, and supported platforms. https://github.com/Alamofire/Alamofire#requirements

## Tags

swift, http-client, networking, urlsession, ios, macos, certificate-pinning, multipart, async-await, apple-platforms, rest-client
