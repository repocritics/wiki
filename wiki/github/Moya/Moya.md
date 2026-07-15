# Moya/Moya

> A Swift network abstraction layer that turns each API into a type-checked enum on top of Alamofire.

[GitHub repo](https://github.com/Moya/Moya) ·
[Official website](https://moya.github.io) ·
[License: MIT](https://github.com/Moya/Moya/blob/master/License.md)

## Overview

Moya is a networking layer for Swift apps, first released in 2014 out of the
Artsy iOS codebase[^1]. Rather than replace Alamofire, it wraps it: you describe
an API as a Swift `enum` conforming to `TargetType`, and Moya turns each case
into a validated `URLRequest`. The pitch is that the ad-hoc "APIManager" /
"NetworkModel" class every team eventually writes is better expressed as a single
enum whose cases are the endpoints, checked by the compiler.

The defining tradeoff is exactly that enum. It buys compile-time safety (you
cannot call an endpoint that does not exist, or forget a path parameter, because
each is an associated value) and makes stubbing trivial — test data is a
first-class property of every target. It costs you scalability: a large API
becomes one enormous enum, and cross-cutting request shape (headers, encoding,
sample data) lives in `switch` statements that grow with every route. Moya is at
its best on small-to-medium APIs and on teams already invested in a reactive
framework; it fights back on very large surfaces.

Moya ships four flavors from one core: the plain callback `Moya`, plus
`RxMoya` (RxSwift), `ReactiveMoya` (ReactiveSwift), and `CombineMoya` (Apple
Combine). Which one you install determines the return type of a request —
`Single<Response>`, `SignalProducer`, or `AnyPublisher` respectively[^2].

## Getting Started

```swift
// Package.swift
.package(url: "https://github.com/Moya/Moya.git", .upToNextMajor(from: "15.0.0"))
```

```swift
import Moya

enum GitHub {
    case zen
    case userProfile(String)
}

extension GitHub: TargetType {
    var baseURL: URL { URL(string: "https://api.github.com")! }
    var path: String {
        switch self {
        case .zen: return "/zen"
        case .userProfile(let name): return "/users/\(name)"
        }
    }
    var method: Moya.Method { .get }
    var task: Task { .requestPlain }
    var headers: [String: String]? { ["Content-Type": "application/json"] }
    var sampleData: Data { Data() }   // used by the built-in test stub
}

let provider = MoyaProvider<GitHub>()
provider.request(.userProfile("ashfurrow")) { result in
    switch result {
    case .success(let response):
        let json = try? response.mapJSON()
    case .failure(let error):   // transport failure only — see below
        print(error)
    }
}
```

## Architecture / How It Works

The core is small. `TargetType` is the contract every endpoint enum implements:
`baseURL`, `path`, `method`, `task`, `headers`, `sampleData`, and an optional
`validationType`. `MoyaProvider<Target: TargetType>` is the request engine,
generic over your enum. When you call `request(_:)`, the provider maps the target
into a `URLRequest`, hands it to Alamofire's `Session`, and wraps the outcome in
a `Response` value carrying the raw `Data`, `statusCode`, and originating
`URLRequest`/`HTTPURLResponse`.

`Task` is where request bodies live: `.requestPlain`, `.requestData`,
`.requestParameters(parameters:encoding:)`, `.requestJSONEncodable`,
`.uploadMultipart`, `.downloadDestination`, and variants. This enum is the seam
between Moya's abstraction and Alamofire's parameter-encoding machinery.

Two extension points matter in practice:

- **`PluginType`** — hooks that fire around every request (`prepare`,
  `willSend`, `didReceive`, `process`). This is how logging
  (`NetworkLoggerPlugin`), auth token injection (`AccessTokenPlugin`), and the
  network activity indicator are implemented without touching your target enum.
- **Stubbing** — a provider constructed with a `stubClosure` (e.g.
  `.immediatelyStub`) returns each target's `sampleData` instead of hitting the
  network. Because sample data is part of `TargetType`, unit tests need no
  mocking framework and no live server.

The reactive variants are thin: `provider.rx.request`, `provider.reactive.request`,
and `provider.requestPublisher` each adapt the same callback core into their
framework's stream type. Signal operators (`filterSuccessfulStatusCodes`,
`mapJSON`, `mapImage`, `mapString`) live alongside them.

## Production Notes

- **4xx/5xx are `.success`, not `.failure`.** A `.failure` result means the
  request never completed a round trip (no connectivity, timeout). A server that
  answers with 404 or 500 is a *successful* response carrying that status code.
  You must call `filterSuccessfulStatusCodes()` / `filterSuccessfulStatusAndRedirectCodes()`
  (or check `response.statusCode`) yourself, or HTTP errors silently pass as
  success. This is the single most common Moya bug.
- **No first-class async/await.** Moya's core API predates Swift Concurrency and
  the primary surface is still callbacks plus the three reactive adapters. Teams
  wanting `try await provider.request(...)` bridge it themselves with
  `withCheckedThrowingContinuation` or through `CombineMoya` +
  `.values`. If your codebase is async/await-native, this friction is permanent,
  not incidental.
- **Enum bloat on large APIs.** One target enum per API means a 60-endpoint
  service becomes a 60-case enum with six parallel `switch` statements. Common
  mitigations: split into multiple providers by domain, or use `MultiTarget` to
  merge them — but `MultiTarget` erases the per-target type and gives back some
  of the compile-time safety that was the point.
- **Alamofire version lock.** Moya sits directly on Alamofire and pins to a major
  version (Moya 15 requires Alamofire 5). You cannot upgrade Alamofire ahead of
  Moya, and a transitive-dependency conflict with another library that wants a
  different Alamofire major will block your build.
- **Reactive dependency weight.** Installing `RxMoya`/`ReactiveMoya` pulls the
  entire RxSwift/ReactiveSwift stack. `CombineMoya` avoids that on Apple
  platforms but requires the Combine framework (Xcode 11.5+ for weak linking).
- **Maintenance cadence.** The library is mature and stable rather than fast-
  moving; the 15.x line has been the current major for a long time and recent
  repository activity is largely CI and dependency housekeeping. Treat it as a
  settled dependency, not one that will grow new capabilities quickly. Interpret
  the README's "actively under development" accordingly.

## When to Use / When Not

**Use when:**
- You want compile-time-checked endpoints and trivially stubbable tests without a
  mocking framework.
- You already use RxSwift, ReactiveSwift, or Combine and want networking that
  returns those stream types directly.
- Your API surface is small to medium and reasonably stable.

**Avoid when:**
- Your codebase is async/await-first and you want native `try await` calls.
- You have a very large or rapidly-changing API where an OpenAPI-generated client
  fits better than a hand-maintained enum.
- You want minimal transitive dependencies — Moya adds Alamofire (and optionally
  a full reactive stack) on top.
- You are doing server-side Swift; use an async HTTP client built for that.

## Alternatives

- Alamofire/Alamofire — the transport Moya wraps; use it directly when you do not
  want the enum abstraction and prefer Alamofire's own request builders.
- kean/Get — modern async/await web client, minimal surface; the natural choice
  when you want Moya's ergonomics but native Swift Concurrency.
- apple/swift-openapi-generator — generate a typed client from an OpenAPI spec
  when the API is large and spec-driven rather than hand-listed.
- swift-server/async-http-client — use for server-side or non-Apple Swift where
  URLSession/Alamofire are not the right base.
- Apple URLSession — zero-dependency baseline with native async/await; reach for
  it when you do not want any third-party networking layer at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-08 | Repository created; extracted from Artsy's iOS app[^1]. |
| 8.x | 2016 | Swift 3 era; reactive extensions (RxSwift/ReactiveSwift) established[^3]. |
| 13.0 | 2019 | Swift 5, Alamofire 4.1+, RxSwift 4[^3]. |
| 14.0 | 2020 | Alamofire 5, RxSwift 5, ReactiveSwift 6; `CombineMoya` added[^2][^3]. |
| 15.0 | ~2021 | Current major. Swift 5.2+, RxSwift 6, ReactiveSwift 6, Alamofire 5[^2]. |

## References

[^1]: Moya README and project history; the library originated in Artsy's iOS
codebase (the Eidolon auction app). https://github.com/Moya/Moya
[^2]: Moya README — installation matrix and reactive extensions
(Moya/RxMoya/ReactiveMoya/CombineMoya), Moya 15.0.0. https://github.com/Moya/Moya/blob/master/Readme.md
[^3]: Moya releases and migration guides. https://github.com/Moya/Moya/releases and https://github.com/Moya/Moya/blob/master/docs/MigrationGuides

## Tags

swift, ios, networking, http-client, alamofire, rxswift, reactiveswift, combine, api-abstraction, testing, cocoapods
