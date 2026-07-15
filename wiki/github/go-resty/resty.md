# go-resty/resty

> A fluent HTTP/REST client for Go that wraps `net/http` with marshaling, retries, and middleware — convenience over the standard library, at the cost of an extra abstraction layer.

[GitHub repo](https://github.com/go-resty/resty) ·
[Official website](https://resty.dev) ·
[License: MIT](https://github.com/go-resty/resty/blob/v3/LICENSE)

## Overview

Resty is a Go client library for making HTTP and REST calls, first tagged in 2015 and stable since the v1 series[^1]. Go's standard `net/http` is complete but verbose: every request means constructing a `*http.Request`, setting headers, checking status codes, reading `resp.Body`, and unmarshaling by hand. Resty collapses that into a chainable `Client` → `Request` → `Response` API where JSON/XML marshaling, query params, retries, and error handling are declared inline. It is one of the most widely used third-party HTTP clients in the Go ecosystem, frequently pulled in by SDKs, API wrappers, and internal service clients.

The library is maintained primarily by its original author, Jeevanandam M. (jeevatkm)[^2]. That single-maintainer model has kept the API coherent and the test coverage high (contributions are required to hold 100% patch coverage[^1]), but it is also the project's main sustainability risk — a genuine bus-factor concern for a dependency this widely embedded.

The defining tension is layering. Resty does not replace `net/http`; it sits on top of `http.Client` and `http.Transport`. You gain a terse API and batteries (retry/backoff, auth schemes, tracing, and in v3 an SSE client and circuit breaker), but you also inherit a second configuration surface, extra allocations per request, and the need to occasionally reach through the wrapper to the underlying transport for TLS, proxy, or connection-pool tuning.

## Getting Started

```bash
# v3 uses the vanity import path resty.dev/v3
go get resty.dev/v3

# v2 (latest stable) uses the GitHub module path
go get github.com/go-resty/resty/v2
```

```go
package main

import (
	"log"

	"resty.dev/v3"
)

type Repo struct {
	FullName string `json:"full_name"`
	Stars    int    `json:"stargazers_count"`
}

func main() {
	client := resty.New()
	defer client.Close() // v3: releases pooled resources

	var repo Repo
	res, err := client.R().
		SetHeader("Accept", "application/json").
		SetResult(&repo).            // auto-unmarshal 2xx body into &repo
		Get("https://api.github.com/repos/go-resty/resty")
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("status=%d %s ★%d", res.StatusCode(), repo.FullName, repo.Stars)
}
```

## Architecture / How It Works

Resty is organized around three long-lived types. A `Client` holds shared configuration: base URL, default headers, auth, the retry policy, registered middleware, and — critically — the underlying `*http.Client` and its transport. A `Request` (obtained via `client.R()`) is a per-call fluent builder that inherits the client's defaults and layers on path params, body, and expected result/error targets. A `Response` wraps `*http.Response` plus the already-read body and timing data.

Requests flow through an ordered chain of **request middlewares** (which serialize the body, apply auth, expand path params, build the `*http.Request`) and, after the call, **response middlewares** (which read the body, run auto-unmarshal into `SetResult`/`SetError` targets, and evaluate retry conditions)[^3]. This middleware model is the real extension point: custom middleware, `OnBeforeRequest`/`OnAfterResponse` hooks, and `OnError` callbacks let you inject logging, metrics, signing, or caching without subclassing.

Because everything ultimately runs on `net/http`, connection pooling, HTTP/2, keep-alives, and TLS behavior come from the standard library's `http.Transport`. Resty exposes conveniences (`SetTimeout`, `SetProxy`, `SetTLSClientConfig`, `SetRedirectPolicy`) but for anything it does not surface you set your own transport via `SetTransport`. Retry is implemented as a loop around the whole request with configurable count, wait, max-wait, backoff with jitter, and `AddRetryCondition` predicates.

v3 is a substantial redesign rather than a patch release. It moves to the `resty.dev/v3` vanity path, adds a first-class SSE (Server-Sent Events) client, a circuit breaker, request/response middleware ordering control, a load-balancer / weighted-round-robin abstraction with SRV-record service discovery, and reworks several v2 APIs[^4]. It requires Go 1.23 or newer[^1].

## Production Notes

**v3 is still a release candidate.** As of mid-2026 the default branch is `v3` and the README points at `resty.dev/v3`, but the newest v3 tag is `v3.0.0-rc.3` (2026-07); the newest *stable* release is `v2.17.2` (2026-02)[^4]. Adopting v3 today means depending on a pre-GA API that can still shift between RCs. For production, many teams still pin v2.

**It is a wrapper, not a replacement.** Under load, the dominant cost is `net/http` and your transport config, not Resty. Default `http.Transport` connection-pool limits (`MaxIdleConnsPerHost` defaults to 2) throttle high-concurrency clients whether or not you use Resty — tune the transport, not just Resty's timeout. Per-request Resty adds allocations (the fluent builder, header copies, body buffering); it is fine for typical API traffic but for extreme throughput hot paths, raw `net/http` with a reused request avoids the overhead.

**Body buffering and auto-unmarshal.** By default Resty reads the entire response body into memory so it can auto-unmarshal and let you re-read it. For large or streaming downloads use `SetDoNotParseResponse(true)` (v2) / the raw-response option and read `res.RawResponse.Body` yourself, or `SetOutputFile` — otherwise you buffer whole payloads. Auto-unmarshal only fires on success codes into `SetResult` and on error codes into `SetError`; forgetting `SetError` means error-body details are silently discarded.

**Shared client state.** A `Client` is safe for concurrent use and *should* be reused (it holds the connection pool). But defaults set on the client (headers, auth, base URL) are inherited by every `R()`; setting request-specific state on the client instead of the request is a common cross-request bleed bug. Create one client per upstream, configure per-call state on the request.

**v2 → v3 migration.** Expect breaking changes: the import path changes, some setter names and signatures moved, and `client.Close()` is now part of the lifecycle. Budget for a real migration pass rather than a version bump. The v1 → v2 jump similarly changed the module path (`gopkg.in` → `github.com/go-resty/resty/v2`).

## When to Use / When Not

**Use when:**
- You make many REST/JSON calls and want marshaling, retries, and error handling declared inline instead of hand-rolled per call.
- You want built-in retry/backoff, request tracing, digest/OAuth-style auth, or (v3) SSE and circuit breaking without assembling libraries yourself.
- You value a consistent, well-tested client API across a codebase or SDK.

**Avoid when:**
- You need maximum throughput on a hot path — raw `net/http` with reused requests avoids the wrapper's allocations.
- Your app makes only a handful of simple calls; the standard library is one dependency fewer.
- You stream large bodies and don't want default in-memory buffering to surprise you.
- You are uncomfortable depending on a pre-GA API (v3) or on a single-maintainer project for critical infrastructure.

## Alternatives

- `net/http` — the standard library; use it directly when your call volume is low or you need full control with zero dependencies.
- `imroc/req` — comparable fluent Go HTTP client with strong built-in request/response dumping and HTTP/3; use when you want similar ergonomics plus deep debug tracing.
- `hashicorp/go-retryablehttp` — a thin retry wrapper over `net/http`; use when retries are all you actually need and you want to stay close to the stdlib.
- `parnurzeal/gorequest` — an older jQuery-style Go HTTP client; use only for legacy code, as it is far less actively maintained.
- `carlmjohnson/requests` — a minimal builder-style stdlib wrapper; use when you want fluency without a large feature surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0 | 2017-09-25 | v1 line stabilized; adopted `go mod` fully by v1.10.0[^1]. |
| v1.12.0 | 2019-02-28 | Final v1 minor before the v2 module split. |
| v2.0.0 | 2019-07-17 | Moved off `gopkg.in` to `github.com/go-resty/resty/v2`[^1]. |
| v2.16.0 | 2024-11-11 | Late-v2 maintenance line. |
| v2.17.2 | 2026-02-14 | Latest stable release. |
| v3.0.0-alpha.1 | 2024-12-01 | Start of the v3 redesign (SSE, circuit breaker, LB)[^4]. |
| v3.0.0-rc.3 | 2026-07-09 | v3 in release-candidate; `resty.dev/v3` vanity path, Go 1.23+[^4]. |

## References

[^1]: Resty README — versioning, minimum Go version, module-path history, and contribution/coverage policy. https://github.com/go-resty/resty
[^2]: Jeevanandam M. (jeevatkm), project creator and primary maintainer. https://github.com/jeevatkm
[^3]: Resty documentation — client/request/response model and middleware chain. https://resty.dev
[^4]: Resty releases — v2.17.2 as latest stable and the v3.0.0 alpha→rc line. https://github.com/go-resty/resty/releases

## Tags

go, golang, http-client, rest-client, sse-client, retry, middleware, api-client, networking, library
