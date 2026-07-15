# httprb/http

> A Ruby HTTP client with a chainable request-building API, native protocol
> handling, and fine-grained timeouts — the "http.rb" gem.

[GitHub repo](https://github.com/httprb/http) ·
[License: MIT](https://github.com/httprb/http/blob/main/LICENSE.txt)

## Overview

http.rb (published as the `http` gem, also called "The Gem!") is a Ruby HTTP
client built around a chainable API modeled after Python's Requests[^1]. The
core idea: configuration methods (`.headers`, `.timeout`, `.auth`, `.follow`)
return a new immutable session object, and you terminate the chain with a verb
(`.get`, `.post`, …) that issues the request. This reads cleanly and keeps
per-request configuration explicit rather than hidden in a global client.

Unlike most Ruby HTTP libraries, http.rb is not a wrapper around the standard
library's `Net::HTTP`. It speaks HTTP/1.1 over raw `TCPSocket` /
`OpenSSL::SSL::SSLSocket` connections itself and delegates only response parsing
to a native extension — currently llhttp, the same parser family used by
Node.js[^2]. Earlier versions depended on the `http_parser.rb` gem (a binding to
Joyent's C parser); the parser has been swapped over the library's lifetime but
the public API has stayed stable.

The defining tradeoff is scope. http.rb does one transport well — HTTP/1.1 with
streaming, persistent connections, and precise timeouts — and deliberately does
not implement HTTP/2, a middleware/adapter abstraction, or cross-thread
connection pooling. Teams that want those either layer other gems on top or pick
a different client. First released as 1.0 in 2015[^3], it is one of the more
mature clients in the ecosystem and is a common dependency for API SDKs.

## Getting Started

```ruby
# Gemfile
gem "http"
```

```ruby
require "http"

# Simplest form — returns an HTTP::Response
response = HTTP.get("https://api.example.com/users")
response.status        # => 200
response.to_s          # => response body as a String

# Chainable configuration; each call returns a new HTTP::Session
json = HTTP.headers(accept: "application/json")
           .timeout(connect: 5, read: 10)
           .auth("Bearer #{token}")
           .get("https://api.example.com/users")
           .parse                    # decodes JSON by Content-Type

# POST a JSON body
HTTP.post("https://api.example.com/users", json: { name: "Ada" })
```

## Architecture / How It Works

The public surface is the `HTTP::Chainable` mixin. Calling a configuration
method on the `HTTP` module (or on a session) constructs an `HTTP::Session`
carrying an options bag (`HTTP::Options`); calling a verb builds an
`HTTP::Request`, hands it to an `HTTP::Client`, and returns an `HTTP::Response`.
Sessions are immutable — each `.headers`/`.timeout`/etc. returns a new session —
which is what makes a configured session safe to share as a template.

Behavior beyond the base request/response cycle is implemented as **features**
(`HTTP::Feature` subclasses) wired in via `.use(...)`: `auto_inflate` /
`auto_deflate` for gzip, `instrumentation`, `logging`, `normalize_uri`, and
`raise_error`. This keeps the hot path small and makes optional behavior opt-in.

**Timeouts** are first-class and layered. You can set a single global timeout or
per-operation `connect` / `read` / `write` values; internally these map to
`HTTP::Timeout::Null`, `Global`, and `PerOperation` strategies wrapping the
socket. This granularity is one of the library's main selling points over
`Net::HTTP`, whose timeout story is coarser.

**Redirects** are handled by `HTTP::Redirector` when `.follow` is set, with a
hop limit and method-rewriting rules per the HTTP spec.

**Persistent connections** (`HTTP.persistent(host)`) keep one keep-alive socket
per origin open across requests, returning a session that reuses the underlying
`HTTP::Client`. Response bodies stream through `HTTP::Response::Body#readpartial`
and are lazily read from the socket.

## Production Notes

- **No timeout by default.** A bare `HTTP.get` will wait indefinitely on a slow
  or hung server. Always set `.timeout(...)` on anything talking to a network
  you don't control — this is the most common production footgun.
- **Persistent sessions are not thread-safe.** A configured (non-persistent)
  session is safe to share because it creates a fresh client per request, but a
  `HTTP.persistent` session pools one socket per origin and must not be shared
  across threads. Wrap it in the `connection_pool` gem for concurrent use.
- **HTTP/1.1 only.** There is no HTTP/2 or HTTP/3 support. Services that require
  or strongly benefit from HTTP/2 multiplexing (many gRPC-adjacent or
  high-fanout APIs) need a different client such as httpx.
- **Streaming bodies must be fully consumed.** If you obtain a response body and
  don't read it to completion (`#readpartial` until `nil`, or `#to_s`), a
  persistent connection can't be safely reused. Close or drain bodies.
- **Native extension build.** The llhttp parser is a compiled extension, so the
  gem needs a working build toolchain (or a precompiled platform gem) at install
  time — occasionally a snag in minimal Alpine/musl containers.
- **TLS verification is on.** Certificates are verified against the system store
  by default; disabling it (`.ssl_context=` with `VERIFY_NONE`) is a deliberate,
  visible act rather than an accidental default.
- **Supported Ruby versions move.** The project explicitly supports only a
  rolling window of recent Ruby releases and will drop older ones at major
  versions, so pin and read `UPGRADING.md` before bumping across a major.

## When to Use / When Not

**Use when:**
- You want a clean, explicit request-building API over `Net::HTTP`'s ergonomics.
- You need precise per-operation timeouts and/or streaming response bodies.
- You're building an API SDK and want a stable, dependency-light HTTP layer.
- Persistent keep-alive connections to a known host matter for throughput.

**Avoid when:**
- You need HTTP/2, HTTP/3, or built-in request multiplexing.
- You want a pluggable adapter/middleware stack (Faraday's model) to swap
  backends or share instrumentation across an org.
- You need concurrent parallel requests out of the box (Typhoeus/libcurl).
- You want zero external gems and `Net::HTTP` already covers your needs.

## Alternatives

- lostisland/faraday — middleware + adapter abstraction; use when you need
  pluggable backends, shared instrumentation, or an org-standard HTTP facade.
- typhoeus/typhoeus — libcurl-backed with parallel/hydra requests; use when you
  need high-concurrency parallel fetches.
- ruby/net-http — the standard library client; use when you want zero
  dependencies and don't need the chainable API or fine timeouts.
- httpx — HTTP/2 and concurrent persistent sessions; use when you need HTTP/2
  multiplexing or built-in connection concurrency.
- excon — minimal, connection-reuse-focused; use when you want a small client
  with explicit socket control (as fog/AWS tooling historically did).

## History

| Version | Date | Notes |
|---------|------|-------|
| (repo)  | 2011-10-06 | Project started by Tony Arcieri[^4]. |
| 1.0.0   | 2015-12-25 | First stable release; chainable API[^3]. |
| 2.0.0   | 2016-04-23 | API cleanup and feature system evolution. |
| 3.0.0   | 2017-10-01 | Major release; caching and internals rework. |
| 4.0.0   | 2018-10-14 | Timeout and connection handling overhaul. |
| 5.0.0   | 2021-05-13 | Modern Ruby baseline; parser/dependency updates. |
| 6.0.0   | 2026-03-17 | Latest major line (6.0.x)[^5]. |

## References

[^1]: http.rb README — "similar to Python's Requests." https://github.com/httprb/http#about
[^2]: llhttp — HTTP parser used by http.rb and Node.js. https://llhttp.org/
[^3]: RubyGems release history for the `http` gem. https://rubygems.org/gems/http/versions
[^4]: httprb/http repository, created 2011-10-06. https://github.com/httprb/http
[^5]: httprb/http releases. https://github.com/httprb/http/releases

## Tags

ruby, http-client, http, rest-client, networking, api-client, streaming, chainable-api, llhttp, gem
