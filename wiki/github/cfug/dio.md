# cfug/dio

> The de facto HTTP client for Dart and Flutter when the standard `package:http` is too thin.

[GitHub repo](https://github.com/cfug/dio) ·
[Official website](https://dio.pub) ·
[License: MIT](https://github.com/cfug/dio/blob/main/LICENSE)

## Overview

dio is an HTTP client library for Dart and Flutter. Where the official
`package:http` is deliberately minimal (send a request, get a response), dio
targets the layer above that: interceptors, `FormData` multipart uploads,
request cancellation, per-request and per-client timeouts, streamed downloads,
response transformers, and swappable transport adapters — all configured
through a single `Dio` instance[^1]. It is the client most Flutter apps reach
for once they outgrow one-off `http.get` calls.

The project was originally authored by @wendux under the @flutterchina
organization, and has been maintained by the Chinese Flutter User Group
(@cfug) since 2023[^2]. That handover coincided with the 5.0 release, which
restructured the codebase into the current monorepo: the core `dio` package
plus first-party plugins for cookies, HTTP/2, native platform transports, and
web[^3].

The defining tension is scope. dio's convenience — global config, an
interceptor chain, automatic JSON decoding — is exactly what makes it heavier
and more opinionated than `package:http`. Teams that want a small, auditable
dependency prefer the standard client; teams that want an axios-style
batteries-included experience prefer dio. It is not a wrapper *over*
`package:http`; it sits directly on `dart:io`'s `HttpClient` (and `fetch` on
web) through its own adapter abstraction.

## Getting Started

```yaml
# pubspec.yaml
dependencies:
  dio: ^5.0.0
```

Or `dart pub add dio` / `flutter pub add dio`.

```dart
import 'package:dio/dio.dart';

final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',
  connectTimeout: const Duration(seconds: 5),
  receiveTimeout: const Duration(seconds: 3),
));

Future<void> main() async {
  try {
    final res = await dio.get<Map<String, dynamic>>('/users/1');
    print(res.data?['name']); // JSON is decoded automatically
  } on DioException catch (e) {
    // e.type distinguishes timeout / bad status / cancel / connection error
    print('${e.type}: ${e.response?.statusCode}');
  }
}
```

## Architecture / How It Works

A `Dio` instance is a façade over three composable pieces:

1. **Interceptors** — an ordered chain run around every request. Each
   `Interceptor` (or `InterceptorsWrapper`) can mutate the request in
   `onRequest`, short-circuit with a cached `Response` in `onResponse`, or
   convert/swallow errors in `onError`. Control flow is explicit via a
   `handler`: `handler.next(...)` continues the chain, `handler.resolve(...)`
   ends it with a response, `handler.reject(...)` ends it with an error.
   `QueuedInterceptor` serializes its callbacks, which is the standard pattern
   for token refresh (hold requests while one refresh is in flight).
2. **HttpClientAdapter** — the transport boundary. The default is
   `IOHttpClientAdapter`, wrapping `dart:io`'s `HttpClient`; on web the
   `dio_web_adapter` uses the browser `fetch`/`XMLHttpRequest`. Swapping the
   adapter is how you get HTTP/2 (`dio_http2_adapter`) or native platform
   stacks (`native_dio_adapter` → Cronet on Android, `NSURLSession` on iOS via
   `cronet_http`/`cupertino_http`)[^3].
3. **Transformer** — encodes request bodies and decodes response bodies.
   The default `BackgroundTransformer` offloads JSON parsing to a background
   isolate so large payloads don't jank the UI thread; `SyncTransformer`
   parses inline.

Options are layered: `BaseOptions` on the client are merged with per-call
`Options`, producing the `RequestOptions` that travel through the chain.
`Response<T>` carries the decoded `data`, headers, and status; `ResponseType`
(`json`, `stream`, `bytes`, `plain`) selects how the body is materialized. A
`CancelToken` passed to one or more requests lets you abort them together.

## Production Notes

**`DioException` is the error surface, not exceptions from `dart:io`.**
Everything failing goes through `DioException` with a `type`
(`connectionTimeout`, `sendTimeout`, `receiveTimeout`, `badResponse`,
`cancel`, `connectionError`, `badCertificate`, `unknown`). By default
`validateStatus` throws on any non-2xx status, so a 404 is a caught exception,
not a returned response — code that expects to inspect `res.statusCode` for
4xx must override `validateStatus` or read `e.response` in the `catch`.

**`FormData` is single-use.** Its underlying streams are consumed on send, so
reusing the same `FormData` for a retry (or in a retry interceptor) throws.
Use `formData.clone()` to build a fresh copy per attempt.

**Timeouts are per-phase, not a total budget.** `connectTimeout`,
`sendTimeout`, and `receiveTimeout` each bound a segment of the request; there
is no single "whole request must finish in N seconds" option. A slow-drip
response that keeps delivering bytes can outlive any individual timeout. Wrap
the future with `Future.any` / a timer if you need a hard ceiling.

**Web is a reduced feature set.** On the web adapter you cannot set some
restricted headers, `HttpClient`-level customization (proxies, bad-cert
callbacks, connection tuning) does not apply, and cross-origin credentials
depend on `withCredentials` plus server CORS. Code that configures the IO
adapter must guard those paths with `kIsWeb`.

**Interceptor ordering and re-entrancy bite.** Interceptors run in
registration order on the way out and reverse on the way back; forgetting to
call `handler.next` silently stalls the request. A refresh interceptor that
issues its own `dio.request(...)` will re-enter the same chain unless it uses a
separate `Dio` instance or a `QueuedInterceptor`, causing infinite loops.

**Upgrades follow a documented migration guide.** dio warns that breaking
changes can land in both major and minor versions, and maintains a Migration
Guide and Compatibility Policy rather than promising SemVer patch-level
stability[^1]. Pin versions and read the changelog before bumping.

## When to Use / When Not

**Use when:**
- You need interceptors (auth, logging, retry, caching) applied globally.
- You upload multipart forms or stream large downloads with progress.
- You want per-request cancellation via `CancelToken`.
- You may need to swap the transport (HTTP/2, native Cronet/NSURLSession) later.

**Avoid when:**
- Your needs are a handful of GETs/POSTs — `package:http` is smaller and audited.
- You want minimal dependency surface / strict SemVer stability.
- You are on web-only and want the thinnest possible `fetch` wrapper.
- You prefer generated, annotation-based API clients over an imperative client.

## Alternatives

- dart-lang/http — the official, minimal Dart HTTP client; no interceptors or `FormData` sugar. Use it when you want a small, well-audited dependency.
- lejard-h/chopper — Retrofit-style, code-generated API client built on `package:http`. Use it when you prefer annotation-driven, type-safe endpoints.
- trevorwang/retrofit.dart — annotation-based client generator that runs *on top of* dio. Use it when you want dio's features but generated call sites.
- flutterchina/dio (legacy) — the pre-2023 org/location; use cfug/dio, which is the maintained continuation.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-04 | Repo created under @flutterchina by @wendux[^2]. |
| 3.x | 2019–2020 | Null-safety pre-work era; widely adopted in Flutter apps. |
| 4.0 | 2021 | Null-safe release. |
| 5.0 | 2023 | Maintenance moved to @cfug; monorepo restructure, plugins split out[^3]. |

## References

[^1]: dio README and package documentation — features, versioning, and Compatibility Policy. https://pub.dev/packages/dio
[^2]: Repository metadata and copyright notice: originally authored by @wendux under @flutterchina, maintained by the Chinese Flutter User Group (@cfug) since 2023. https://github.com/cfug/dio
[^3]: dio monorepo plugin packages — `dio_cookie_manager`, `dio_http2_adapter`, `native_dio_adapter`, `dio_web_adapter`, `dio_compatibility_layer`. https://github.com/cfug/dio/tree/main/plugins

## Tags

dart, flutter, http-client, networking, interceptors, formdata, rest-api, mobile, cross-platform, cancellation
