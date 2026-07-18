# yesodweb/wai

> Haskell's Web Application Interface — the WSGI/Rack of Haskell — plus Warp,
> the server that runs nearly every Haskell web app in production.

[GitHub repo](https://github.com/yesodweb/wai) ·
[Hackage docs](https://hackage.haskell.org/package/wai) ·
License: MIT

## Overview

WAI (Web Application Interface) is the contract between Haskell web frameworks
and web servers, in the same role WSGI plays for Python and Rack for Ruby: an
application is a function from a request to a response, and any server that
speaks WAI can run any framework that targets it[^1]. The repo dates to January
2010 and is maintained by Michael Snoyman (Yesod) with Kazu Yamamoto driving
the server and HTTP/2/3 work. Its 875 stars understate its position badly —
Yesod, Servant, Scotty, Spock, and IHP all compile down to WAI applications,
so this repo sits under essentially the whole Haskell web ecosystem. It is
actively maintained (pushed within the last week as of mid-2026).

The catch: this is not one library but a mono-repo of ~14 packages. `wai`
itself is a tiny, deliberately boring interface package on the 3.x series
since 2014. The bulk of the code — and the churn — is **Warp**, the flagship
HTTP server, plus `warp-tls`, `wai-extra` (middleware grab-bag),
`wai-app-static`, `wai-websockets`, and infrastructure packages
(`time-manager`, `recv`, `mime-types`). The defining tradeoff: raw WAI is
low-level by design — no routing, no sessions, no templating. Most users
should reach it through a framework and drop down to WAI only for middleware
and deployment.

## Getting Started

```bash
cabal init && cabal build
# add to build-depends: wai, warp, http-types
```

```haskell
{-# LANGUAGE OverloadedStrings #-}
import Network.Wai
import Network.Wai.Handler.Warp (run)
import Network.HTTP.Types (status200)

main :: IO ()
main = run 8080 app

app :: Application
app _req respond =
  respond $ responseLBS status200
    [("Content-Type", "text/plain")]
    "Hello, WAI!"
```

## Architecture / How It Works

The core type is continuation-passing style, adopted in wai 3.0 (2014)[^2]:

```haskell
type Application =
  Request -> (Response -> IO ResponseReceived) -> IO ResponseReceived
```

The continuation guarantees resource cleanup (earlier versions leaned on
`enumerator`, then `conduit`, for streaming; 3.0 removed the conduit dependency
from the core and made bracketing explicit). `Middleware` is just
`Application -> Application`, which is why WAI middleware composes trivially —
gzip, request logging, forced SSL, and auth from `wai-extra` are all plain
function wrappers.

**Warp** is where the engineering lives[^3]. It spawns one GHC green thread per
connection and relies on GHC's event-driven IO manager to multiplex them, so
"thread per connection" scales to tens of thousands of connections without OS
threads. Notable internals: a shared timeout manager (one timer thread instead
of per-connection timers, doubling as slowloris protection), `sendfile`-based
static file responses, and careful buffer reuse in the `recv` package. HTTP/2
is delegated to Kazu Yamamoto's `http2` package and negotiated automatically
via ALPN when running under `warp-tls`[^4]. An experimental QUIC/HTTP-3 backend
lives in `warp-quic`.

Request bodies are pull-based: `getRequestBodyChunk` returns the next chunk
and can only be consumed once. Middleware that needs to inspect the body must
buffer and replay it — a recurring source of subtle bugs in logging and
signature-verification middleware.

## Production Notes

- **Runtime flags matter.** `-threaded` at compile time, `+RTS -N` at runtime.
  Without them Warp runs, silently, on one core — the classic first-deploy
  mistake.
- **Timeout manager defaults.** Warp times out inactive connections (30s
  default). Long-polling endpoints and slow uploads need `setTimeout` or a
  per-request `pauseTimeout`.
- **Exceptions.** Warp catches per-connection exceptions and returns a generic
  500; customize with `setOnException` / `setOnExceptionResponse`. There is no
  structured error reporting by default — wire your own.
- **TLS strategy.** `warp-tls` is fine in-process, but most production
  deployments still terminate TLS at nginx/HAProxy/a load balancer and speak
  plain HTTP to Warp. If you do that, HTTP/2 to the app goes away (h2c is not
  the default path) and you need `wai-extra`'s forwarded-header middleware for
  correct client IPs.
- **HTTP/2 surprises.** Enabled automatically over ALPN — apps that assumed
  HTTP/1.1 semantics (e.g., relying on connection close behavior) have been
  bitten after simply upgrading warp-tls.
- **Stability profile.** The `wai` interface is remarkably stable — 3.x since
  2014, breaking changes rare and telegraphed via deprecation (`requestBody` →
  `getRequestBodyChunk`). Warp minor releases are frequent; releases are
  per-package tags (`warp-tls-3.4.x` etc.), so pin per package.
- **Performance.** Warp benchmarks near the top of native-code HTTP
  servers[^3]; in practice application logic and the database dominate long
  before Warp does. Don't pick Haskell for Warp's throughput alone.

## When to Use / When Not

**Use when:**
- You're deploying any Haskell web framework — you're already using it;
  learning WAI's `Middleware` layer pays off immediately.
- You need a small, stable HTTP surface for a Haskell service without a
  framework (internal APIs, webhooks, health endpoints).
- You want HTTP/2 (and experimental HTTP/3) in Haskell without writing
  protocol code.

**Avoid when:**
- You want batteries included — raw WAI has no routing or sessions; use
  Scotty/Servant/Yesod on top instead of reimplementing them.
- You're not writing Haskell; the interface pattern is the same as WSGI/Rack
  but nothing here is reusable outside GHC.
- You're committed to the Snap ecosystem, which has its own server and API.

## Alternatives

- snapframework/snap-server — the other production Haskell HTTP server, with
  its own (non-WAI) API; use it if you're building on Snap.
- Happstack/happstack-server — self-contained Haskell web stack predating
  WAI's dominance; mostly of interest to existing Happstack codebases.
- rack/rack — Ruby's equivalent interface; the conceptual counterpart when
  you're not in Haskell.
- pallets/werkzeug — Python WSGI toolkit occupying the same "interface +
  utilities" niche.

## History

| Version | Date | Notes |
|---------|------|-------|
| wai 0.x | 2010-01 | Initial release, enumerator-based streaming; repo created 2010-01-17. |
| wai 1.0 | 2012 | conduit-based streaming interface. |
| wai 3.0 | 2014 | CPS `Application` type; conduit removed from core[^2]. |
| warp 3.1 | 2015 | HTTP/2 support via the `http2` package + ALPN[^4]. |
| warp-quic | ~2021 | Experimental QUIC/HTTP-3 backend added to the repo. |
| warp-tls 3.4.14 | 2026 | Latest tagged package release line as of mid-2026. |

## References

[^1]: Yesod book, "Web Application Interface". https://www.yesodweb.com/book/web-application-interface
[^2]: `wai` changelog on Hackage. https://hackage.haskell.org/package/wai/changelog
[^3]: Kazu Yamamoto, Michael Snoyman, Andreas Voellmy, "Warp", The Performance of Open Source Applications. https://aosabook.org/en/posa/warp.html
[^4]: `warp` package documentation. https://hackage.haskell.org/package/warp

## Tags

haskell, web-server, http, server-interface, middleware, warp, http2, streaming, wsgi-analog, yesod
