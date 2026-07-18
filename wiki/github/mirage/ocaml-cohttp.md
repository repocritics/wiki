# mirage/ocaml-cohttp

> HTTP/1.1 client and server library for OCaml — one portable parser,
> pluggable Lwt / Async / Eio / curl / js_of_ocaml backends.

[GitHub repo](https://github.com/mirage/ocaml-cohttp) ·
[API docs](https://mirage.github.io/ocaml-cohttp/) ·
[License: ISC](https://github.com/mirage/ocaml-cohttp/blob/main/LICENSE.md)

## Overview

Cohttp is the longest-serving HTTP library in the OCaml ecosystem — the repo
dates to 2009, predating opam itself — and grew up inside the MirageOS
unikernel project[^1]. That origin explains its defining design decision:
protocol logic is written against an abstract `IO` signature, so the same
request/response code runs on Unix with Lwt or Jane Street's Async, inside a
MirageOS unikernel with no OS at all, in the browser via js_of_ocaml (mapped
to XMLHttpRequest), over libcurl, or on OCaml 5's effects-based Eio[^2]. Since
the v6 line, the core types and parser live in a deliberately dependency-free
`http` package that other OCaml HTTP projects can share[^3].

The tradeoff for that portability is a lowest-common-denominator feature set.
Cohttp is HTTP/1.1 only — no HTTP/2 or HTTP/3 — and deliberately omits
redirect following[^4], cookie jars, sessions, and multipart handling,
delegating all of them to user code or third-party libraries. It is a
protocol library, not a web framework.

The 774 stars materially understate its position: absolute star counts in the
OCaml world are small, and cohttp has been the default "make an HTTP request"
answer for OCaml applications for over a decade, sitting near the root of a
large opam reverse-dependency tree. Maintenance is real but unhurried — v6.0.0
spent two years in alpha/beta (2022-10 to 2024-11)[^3], and the repo's last
push (April 2026) trails months behind at any given time. It is maintained
infrastructure, not a fast-moving project.

## Getting Started

Install the backend(s) you need from opam — `cohttp` alone is just the core:

```bash
opam install cohttp-lwt-unix    # or: cohttp-async, cohttp-eio, cohttp-curl-lwt
```

A minimal Lwt client (from the README, still the canonical example)[^1]:

```ocaml
open Lwt
open Cohttp
open Cohttp_lwt_unix

let main =
  Client.get (Uri.of_string "https://example.com/") >>= fun (resp, body) ->
  let code = resp |> Response.status |> Code.code_of_status in
  Printf.printf "Response code: %d\n" code;
  body |> Cohttp_lwt.Body.to_string

let () = print_endline (Lwt_main.run main)
```

Build with dune by declaring `(libraries cohttp-lwt-unix)` in the executable
stanza. A server is a function of type
`conn -> Http.Request.t -> Cohttp_lwt.Body.t -> (Http.Response.t * Cohttp_lwt.Body.t) Lwt.t`
passed to `Server.make`.

## Architecture / How It Works

The package layout mirrors the abstraction stack[^2]:

- **`http`** — request/response/header types plus a fast HTTP/1.1 parser,
  zero dependencies. Introduced in the v6 alpha series so competing OCaml
  HTTP stacks can interoperate on shared types[^3].
- **`cohttp`** — backend-independent logic (status codes, header utilities,
  accept-header parsing) on top of `http`.
- **Backend packages** — `cohttp-lwt` (OS-independent Lwt, used by
  `cohttp-mirage` for unikernels), `cohttp-lwt-unix`, `cohttp-async`,
  `cohttp-eio` (OCaml 5 / effects), `cohttp-curl-{lwt,async}` (libcurl via
  ocurl), and `cohttp-lwt-jsoo` (browser XHR). Implementing the `IO`
  signature in `lib/s.mli` yields a new backend.

Connection establishment and TLS are delegated to a sibling MirageOS
library, **conduit**. This is a real coupling story: which TLS stack you get
(`ocaml-tls` vs. OpenSSL-based `lwt_ssl`) depends on what happens to be
installed, with `lwt_ssl` winning by default when both are present and a
`CONDUIT_TLS=native` environment variable to force the pure-OCaml stack[^1].
An attempt to adopt conduit 3.0's redesigned API in 2020 was released as
v3.0.0, then aborted and reverted for v4.0.0 when the design discussion
failed to reach consensus[^3] — a rare public example of a rolled-back major
version.

Bodies are streams, not strings: `Cohttp_lwt.Body.t` may be backed by an
unconsumed connection, which is why draining is the caller's responsibility
(see below). The v5.0.0 header rewrite replaced a map with an ordered
associative list, preserving transmission order per RFC 7230 §3.2.2, and
changed `get` to return only the last value of a duplicated header[^5].

## Production Notes

- **Drain your bodies.** If you do not consume or explicitly
  `Body.drain_body` a response body, the underlying connection leaks. This
  is the number-one cohttp footgun; the library now logs a warning when it
  detects a leaked body[^6], and the README's own redirect example exists
  largely to demonstrate correct draining.
- **HTTP/1.1 only.** No HTTP/2, no HTTP/3, no server push. If a peer or
  load balancer requires h2, cohttp is the wrong layer — use piaf or h2
  directly, or terminate at a proxy.
- **Batteries not included.** Redirect following is intentionally absent
  (see issue #76 for the rationale)[^4]; cookies/sessions need
  `ocaml-session`; multipart needs `multipart_form` or similar. Budget for
  assembling these yourself.
- **You pick a concurrency runtime first.** Lwt, Async, and Eio ecosystems
  do not mix in one process. Choosing `cohttp-async` binds you to the Jane
  Street stack; `cohttp-eio` requires OCaml 5. Library authors targeting
  cohttp usually functorize or ship per-backend packages, which multiplies
  maintenance.
- **Upgrade pains are semantic, not syntactic.** v5.0.0 changed
  `Header.get` semantics (last value only, no implicit concatenation) and
  stopped lowercasing header keys in favor of case-insensitive
  comparison[^5]. v6.0.0 removed the scheme field from requests, so
  `Request.uri` no longer round-trips the URI a request was made with[^3].
  Both broke code silently rather than at compile time.
- **Server performance is adequate, not leading.** The classic
  `Cohttp_lwt_unix.Server` loses benchmarks to httpaf-family servers; the v6
  parser rewrite narrowed the gap and the minimal `cohttp-server-lwt-unix`
  package exists specifically as a leaner server path[^3]. For high-QPS
  services, measure before committing.
- 106 open issues against a small active maintainer group; PRs get reviewed,
  but expect latency measured in weeks.

## When to Use / When Not

**Use when:**
- You need HTTP in an OCaml project and want the most widely deployed,
  best-documented option with a decade of production history.
- You target MirageOS unikernels or js_of_ocaml — cohttp is effectively the
  only game in town for backend-portable HTTP.
- You want to share the dependency-free `http` types with other libraries.
- You need client and server from one coherent API across Lwt/Async/Eio.

**Avoid when:**
- You need HTTP/2 or HTTP/3 (piaf, h2, or a fronting proxy instead).
- You want a web framework with sessions, templating, and WebSockets out of
  the box — that is dream, not cohttp.
- Raw server throughput is the requirement; httpaf-lineage servers are
  faster and stricter about protocol conformance.
- Your team is not already committed to an OCaml concurrency runtime; the
  backend choice is a bigger decision than the HTTP library itself.

## Alternatives

- aantron/dream — batteries-included OCaml web framework (sessions,
  WebSockets, TLS, templating); use it when you want a framework rather
  than a protocol library.
- inhabitedtype/httpaf — stricter, faster HTTP/1.1 state machine; use it
  for high-throughput servers (note: most active development now lives in
  anmonteiro's fork).
- anmonteiro/piaf — client library with HTTP/2 support built on the
  httpaf/h2 stack; use it when you need h2 or a higher-level client.
- ocsigen/ocsigenserver — full-featured OCaml web server with modules
  (static files, auth, CGI); use it when you want a configured server, not
  a library.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.99.0 | 2017-07 | Split into per-backend opam packages; jbuilder build[^7]. |
| 1.0.0 | 2017-11 | Stable release; wrapped top-level modules. |
| 3.0.0 | 2020-11 | Conduit 3.0 adoption — released, then aborted[^3]. |
| 4.0.0 | 2021-03 | Conduit 3.0 changes reverted; body-leak warnings. |
| 5.0.0 | 2021-12 | Header rewrite: ordered assoc list, `get` semantics change[^5]. |
| 6.0.0-alpha0 | 2022-10 | Dependency-free `http` package, Eio + curl backends, new parser[^3]. |
| 6.0.0 | 2024-11 | v6 stable after two years of alphas; cohttp-eio for OCaml 5[^3]. |
| 6.1.0 | 2025-03 | Incremental fixes. |
| 6.2.0 | 2025-12 | Latest release line. |

## References

[^1]: ocaml-cohttp README. https://github.com/mirage/ocaml-cohttp#readme
[^2]: Backend list and `IO` signature, README. https://github.com/mirage/ocaml-cohttp#ocaml-cohttp----an-ocaml-library-for-http-clients-and-servers
[^3]: CHANGES.md — v4.0.0, v5.0.0, v6.0.0~alpha0, v6.0.0 entries. https://github.com/mirage/ocaml-cohttp/blob/main/CHANGES.md
[^4]: Issue #76 — rationale for not following redirects in the API. https://github.com/mirage/ocaml-cohttp/issues/76
[^5]: CHANGES.md v5.0.0 — new Header implementation and semantics. https://github.com/mirage/ocaml-cohttp/blob/main/CHANGES.md
[^6]: Issue #730 — leaked body warning. https://github.com/mirage/ocaml-cohttp/issues/730
[^7]: CHANGES.md v0.99.0 — package split announcement. https://github.com/mirage/ocaml-cohttp/blob/main/CHANGES.md

## Tags

ocaml, http, http-client, http-server, lwt, async, eio, mirageos, unikernel, networking
