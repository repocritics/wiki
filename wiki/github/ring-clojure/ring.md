# ring-clojure/ring

> Clojure's HTTP abstraction: a request is a map, a response is a map, a handler is a function, and everything else is middleware.

[GitHub repo](https://github.com/ring-clojure/ring) ·
[SPEC](https://github.com/ring-clojure/ring/blob/master/SPEC.md) ·
[License: MIT](https://github.com/ring-clojure/ring/blob/master/LICENSE)

## Overview

Ring is the low-level HTTP contract that nearly the entire Clojure web ecosystem
is built on. It was created by Mark McGranaghan in 2009, modeled directly on
Ruby's Rack and Python's WSGI, and has been maintained for most of its life by
James Reeves (weavejester)[^1]. The idea is deliberately small: an HTTP request
is an ordinary Clojure map (`:request-method`, `:uri`, `:headers`, `:body`, …),
an HTTP response is another map (`:status`, `:headers`, `:body`), a handler is a
plain function from request to response, and middleware is a function that wraps
a handler to return a new handler. That is essentially the whole model, and it
is specified formally in `SPEC.md` rather than enforced by types[^2].

Because the contract is just maps and functions, Ring is less a framework than a
substrate. Routing, templating, auth, and content negotiation are not part of
core Ring — they are separate libraries (Compojure, reitit, ring-defaults, Buddy)
that agree to speak the Ring request/response format. This is Ring's defining
tension: the core is stable and minimal to the point of being austere, and any
real application is assembled from a stack of third-party middleware whose
quality and maintenance vary. You get composability and near-total decoupling
from any one web server; you do not get an out-of-the-box application skeleton.

## Getting Started

`deps.edn`:

```clojure
{:deps {ring/ring-core          {:mvn/version "1.15.5"}
        ring/ring-jetty-adapter {:mvn/version "1.15.5"}}}
```

```clojure
(require '[ring.adapter.jetty :refer [run-jetty]]
         '[ring.middleware.params :refer [wrap-params]])

(defn handler [request]
  {:status  200
   :headers {"Content-Type" "text/plain"}
   :body    (str "Hello, " (get-in request [:params "name"] "World"))})

;; middleware is just function composition
(def app (wrap-params handler))

(run-jetty app {:port 3000 :join? false})
```

A handler is any `fn` returning a response map; `wrap-params` is a higher-order
function that parses the query/body params into `:params` before calling the
inner handler. Add routing (Compojure/reitit) on top when one handler is not
enough.

## Architecture / How It Works

Ring is split into small libraries so applications only pull in what they use[^3]:

- `ring/ring-core` — the request/response helpers and the bundled middleware
  (params, cookies, sessions, multipart, content-type, not-modified, …).
- `org.ring-clojure/ring-core-protocols` — just the `StreamableResponseBody`
  protocol, extracted in 1.12.0 so libraries can produce response bodies without
  depending on all of core[^4].
- `ring/ring-jetty-adapter` — an embedded Jetty server that turns Ring handlers
  into a running HTTP listener. This is the reference adapter.
- `ring/ring-servlet` (legacy `javax`, Servlet ≤ 4.0) and
  `org.ring-clojure/ring-jakarta-servlet` (Servlet ≥ 5.0) — deploy a handler
  into an external application server.
- `ring/ring-devel` — reload and stacktrace middleware for development only.

Handlers come in two shapes. The **synchronous** handler is `(fn [request] …)`
returning a response map. The **asynchronous** handler is a three-arity
`(fn [request respond raise] …)` that calls `respond` with a response or `raise`
with an exception; this was added in Ring 1.6 so a request can be held without
tying up a thread[^5]. Middleware must be written to support whichever arities
it wraps — a lot of community middleware only implements the synchronous path.

Response bodies are polymorphic via the `StreamableResponseBody` protocol: a
`String`, `ISeq`, `File`, or `InputStream` each know how to write themselves to
the output stream. WebSockets were added as experimental in 1.11 (2023) via a
separate `ring-websocket-protocols` library and an upgrade-request convention,
so the same handler abstraction covers both request/response and socket
upgrades[^6].

## Production Notes

**Core Ring ships no security defaults.** A bare handler has no CSRF protection,
no security headers, no sane cookie flags. The near-universal fix is
`ring/ring-defaults` (`site-defaults` / `api-defaults` / `secure-site-defaults`),
which is a *separate* library. Deploying `ring-core` alone and assuming it is
safe is a recurring mistake.

**The Jetty adapter is thread-per-request by default.** Each in-flight request
holds a worker thread across blocking I/O. High-concurrency or long-poll
workloads either need the async (three-arity) handler path — and async-aware
middleware — or a different server (http-kit, Aleph) built around non-blocking
I/O from the start. Tuning knobs like `:acceptor-threads` and `:selector-threads`
exist on the Jetty adapter but do not change the blocking model.

**The servlet split matters for deployment.** Application servers moved the
namespace from `javax.servlet` to `jakarta.servlet`. Ring keeps two adapters:
`ring-servlet` for the old `javax` world and `ring-jakarta-servlet` for Jakarta
EE 9+. Picking the wrong one against your container yields `ClassNotFoundException`
at deploy time, not compile time.

**The default session store is in-memory.** It does not survive a restart and is
not shared across instances, so horizontally-scaled deployments need a cookie
store (signed) or an external store; otherwise sessions vanish on the next
deploy or land on the wrong node behind a load balancer.

**The request `:body` is a one-shot `InputStream`.** Read it once. Middleware
ordering is load-bearing — `wrap-params` must run before code that reads
`:params`, and body-consuming middleware placed twice will find an exhausted
stream. A 2026 release (1.15.5) also patched a ReDoS vector in
`wrap-nested-params`, so keeping current on point releases has a real security
dimension[^7].

**Java baseline is climbing.** As of 1.14, `ring-core` requires Java 11+ and the
Jetty adapter requires Java 17+. Upgrading Ring can force a JVM upgrade.

## When to Use / When Not

**Use when:**
- You are building anything HTTP in Clojure and want the standard interchange
  format every other library already speaks.
- You want to swap web servers (Jetty ↔ servlet container ↔ http-kit) without
  rewriting handlers.
- You prefer assembling a stack from explicit middleware over adopting a
  batteries-included framework.

**Avoid / look elsewhere when:**
- You want routing, auth, and validation out of the box — Ring gives you none of
  that; reach for reitit or a fuller framework on top.
- Your workload is highly concurrent or streaming-first and you don't want the
  blocking thread-per-request default — an async-native server fits better.
- You are not in the JVM/Clojure ecosystem at all — Ring is Clojure-specific.

## Alternatives

- metosin/reitit — data-driven routing and coercion layered *on top of* Ring;
  use it with Ring, not instead of it.
- weavejester/compojure — the original small routing DSL over Ring handlers;
  use when you want minimal routing and nothing else.
- pedestal/pedestal — use when you prefer an interceptor queue (async-friendly,
  data-oriented) over Ring's function-wrapping middleware.
- http-kit/http-kit — use when you want a lightweight non-blocking server with
  built-in async and WebSockets (it also speaks the Ring handler shape).
- clj-commons/aleph — use when you need Netty-grade streaming and backpressure
  via Manifold rather than blocking request/response.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2009 | Created by Mark McGranaghan; modeled on Rack/WSGI[^1]. |
| 1.6.0 | 2017 | Asynchronous three-arity handler support[^5]. |
| 1.9.0 | 2021-02-03 | SSL/cipher options and async-response performance work. |
| 1.11.0 | 2023-12-25 | Experimental WebSockets + `ring-jakarta-servlet` adapter[^6]. |
| 1.12.0 | 2024-03-11 | `ring.core.protocols` extracted to its own library[^4]. |
| 1.14.0 | 2025-03-25 | Java 11+ (core) / Java 17+ (Jetty); Jetty 12; WS tuning options. |
| 1.15.0 | 2025-09-12 | `wrap-content-length` middleware; Jetty callback fixes. |
| 1.15.5 | 2026-06-23 | ReDoS fix in `wrap-nested-params`[^7]. |

## References

[^1]: Ring README — copyright "2009–2026 Mark McGranaghan, James Reeves & contributors." https://github.com/ring-clojure/ring
[^2]: Ring SPEC.md — the formal request/response/handler/middleware contract. https://github.com/ring-clojure/ring/blob/master/SPEC.md
[^3]: Ring README, "Libraries" — the ring-core / adapters / servlet / devel split. https://github.com/ring-clojure/ring#libraries
[^4]: Ring CHANGELOG 1.12.0 (2024-03-11) — "Moved ring.core.protocols into its own library." https://github.com/ring-clojure/ring/blob/master/CHANGELOG.md
[^5]: Ring wiki, "Concepts" / async handlers (three-arity respond/raise). https://github.com/ring-clojure/ring/wiki/Concepts
[^6]: Ring CHANGELOG 1.11.0-alpha1 (2023-08-17) — "Added experimental websocket support"; "Added ring-jakarta-servlet project." https://github.com/ring-clojure/ring/blob/master/CHANGELOG.md
[^7]: Ring CHANGELOG 1.15.5 (2026-06-23) — "Fixed ReDoS attack vector in `wrap-nested-params` middleware." https://github.com/ring-clojure/ring/blob/master/CHANGELOG.md

## Tags

clojure, http, web-server, middleware, jvm, http-abstraction, ring, jetty, servlet, backend, async
