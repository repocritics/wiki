# lostisland/faraday

> A Ruby HTTP client that abstracts over swappable backends and processes each request through a Rack-style middleware stack.

[GitHub repo](https://github.com/lostisland/faraday) ·
[Official website](https://lostisland.github.io/faraday) ·
[License: MIT](https://github.com/lostisland/faraday/blob/main/LICENSE.md)

## Overview

Faraday is an HTTP client abstraction layer for Ruby, first released in 2009[^1]. Its
premise is that the code that makes an HTTP call should not be coupled to *how* the call
is made. You write against one `Faraday::Connection` API; the actual transport — `Net::HTTP`,
Typhoeus, Patron, Excon, HTTPClient, `async-http` — is a pluggable adapter chosen at
configuration time. This is why a large share of Ruby API-client gems (Octokit and many
others) are built on Faraday: the library author gets middleware and adapter-swapping for
free and lets the consuming app pick its backend.

The second defining idea is borrowed directly from Rack. Every request passes through an
ordered stack of middleware wrapping a single adapter at the bottom. Request middleware can
mutate the outgoing request (encode a body, set auth headers); response middleware can
transform the result (parse JSON, raise on 4xx/5xx, follow redirects, log). The same mental
model as Rack app middleware, applied to the client side.

The defining tension is that this flexibility is also a configuration burden. Faraday does
very little by default — it does not parse response bodies, does not raise on HTTP error
status, and (since 2.x) does not even bundle most adapters. Nearly every non-trivial feature
is a middleware you must add, often from a separate gem. The library trades batteries-included
convenience for composability, and the 1.x→2.x transition made that trade sharper by
extracting adapters and middleware into their own gems[^2].

## Getting Started

```ruby
# Gemfile
gem "faraday"
gem "faraday-retry"   # extracted to its own gem in Faraday 2.0
```

```ruby
require "faraday"
require "faraday/retry"

conn = Faraday.new(url: "https://api.example.com") do |f|
  f.request  :json                 # encode request bodies as JSON
  f.request  :retry, max: 2        # from faraday-retry
  f.response :json                 # parse response bodies as JSON
  f.response :raise_error          # raise on 4xx/5xx (off by default)
  f.adapter  Faraday.default_adapter   # Net::HTTP unless overridden
end

resp = conn.post("/posts", { title: "hello" })
resp.status   # => 201
resp.body     # => parsed Hash, because response :json is in the stack
```

Without `response :json` the body is a raw string; without `response :raise_error` a 500
returns normally with `resp.status == 500`. Both are deliberate defaults, and both surprise
newcomers.

## Architecture / How It Works

A `Faraday::Connection` holds a **middleware stack** built by `Faraday::RackBuilder`. At
request time Faraday constructs an `env` (a mutable struct carrying method, URL, request
headers, body, and later the response) and threads it through the stack:

1. **Request middleware** run top-to-bottom, transforming the outgoing `env` (JSON/URL
   encoding, `Authorization` headers, multipart bodies, instrumentation).
2. **The adapter** — exactly one, always at the bottom — performs the actual network I/O and
   writes status, response headers, and body back into `env`.
3. **Response middleware** unwind bottom-to-top, transforming the result (body parsing,
   `raise_error`, redirect following, logging).

Adapters are thin wrappers that translate `env` into a call on the underlying HTTP library
and back. Because the adapter is isolated at one layer, switching from `Net::HTTP` to
Typhoeus is a one-line change with no effect on the rest of your client code — the central
payoff of the whole design.

**Parallel requests** are an adapter capability, not a Faraday-core feature. Only adapters
with an async backend (historically Typhoeus via libcurl's multi interface, `em-http`)
support `conn.in_parallel { ... }`; the default `Net::HTTP` adapter runs sequentially and
silently ignores the parallel block semantics.

The pivotal architectural event was the **gem extraction** across 1.0 (2020) and 2.0 (2022).
Before it, Faraday shipped adapters for many HTTP libraries and middleware like retry and
multipart inside the main gem. After it, the core gem carries the `Net::HTTP` adapter plus a
small set of middleware, and everything else — `faraday-retry`, `faraday-typhoeus`,
`faraday-excon`, `faraday-net_http_persistent`, `faraday-multipart`, and so on — lives in
independent, separately versioned gems[^2]. This shrank the core and decoupled release
cycles, at the cost of a discovery problem: you now have to know which gem provides the
piece you want.

## Production Notes

**The 1.x → 2.0 upgrade is the dominant operational story.** Code that worked on Faraday 1
routinely breaks on 2 not because APIs changed but because a middleware or adapter is no
longer bundled. `Faraday::Request#retry`, multipart uploads, and non-`Net::HTTP` adapters all
became separate gems you must add to the Gemfile[^2]. Faraday 1.x emitted deprecation
warnings pointing at these; upgrading means reading them, not just bumping the version.

**Silent defaults are footguns.** No body parsing and no error raising unless you add the
middleware. Teams frequently ship code that treats a 500 as success because `raise_error`
was never in the stack, or that calls `JSON.parse(resp.body)` manually and then double-parses
once someone adds `response :json`.

**Middleware order is load-bearing and easy to get wrong.** `raise_error` should generally sit
*after* the parsing middleware in declaration order so error responses are still parsed;
retry should wrap the adapter; logging placement changes whether you see raw or transformed
bodies. There is no validation — a mis-ordered stack simply behaves subtly wrong.

**Adapter choice has real consequences.** `Net::HTTP` is the safe default but has no built-in
connection pooling across threads; high-throughput services often move to
`net-http-persistent` or Typhoeus. Persistent/keep-alive connections and their thread-safety
characteristics are the adapter's responsibility, not Faraday's — read the specific adapter
gem's docs, because the abstraction does not paper over these differences.

**Version skew across the ecosystem.** Because adapters and middleware are now separate gems,
a single app can end up with incompatible combinations (a middleware gem written for the 1.x
middleware interface used under Faraday 2). Pin and test the whole set together; the core
version alone does not tell you the app is consistent.

**Timeouts** are passed through to the adapter (`conn.options.timeout`,
`conn.options.open_timeout`) and honored only as well as the underlying library honors them.
Behavior on DNS timeouts and read timeouts differs per adapter.

## When to Use / When Not

**Use when:**
- You are writing a reusable API-client gem and want consumers to pick their own HTTP backend.
- You need request/response middleware — auth, retry, instrumentation, parsing — composed cleanly.
- You want to swap or A/B different HTTP backends without touching call sites.
- You are already in an ecosystem (Octokit, many SDKs) that expects a Faraday connection.

**Avoid when:**
- You just need to fetch a URL in a script; `Net::HTTP` or `http.rb` is less ceremony.
- You want body parsing, error raising, and retries on by default without assembling a stack.
- You need first-class concurrent requests; a natively async client (`httpx`, `async-http`) fits better than Faraday's adapter-dependent parallel mode.
- You dislike the multi-gem model and want one dependency that includes everything.

## Alternatives

- httprb/http — "http.rb", a chainable client with sensible defaults (parses, raises); simpler when you don't need pluggable backends or middleware.
- typhoeus/typhoeus — libcurl-based; use directly when parallel/concurrent requests are the primary need rather than a Faraday add-on.
- excon/excon — low-overhead client with built-in connection reuse; use when you want a single fast adapter without an abstraction layer.
- HoneyryderChuck/httpx — modern client with native HTTP/2, connection pooling, and concurrency; use when you want async as a core feature, not an adapter trait.
- jnunemaker/httparty — convenience wrapper for quick JSON API consumption; use for scripts and small integrations where flexibility doesn't matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2009-12 | Initial release; middleware-over-adapters concept[^1]. |
| 0.9 | 2014 | Long-lived 0.x line; adapters and middleware bundled in-core. |
| 1.0 | 2020-01 | Begins extracting adapters/middleware to external gems; deprecation warnings[^2]. |
| 2.0 | 2022-01 | Completes extraction — core ships `Net::HTTP` adapter + base middleware only; drops older Rubies[^2]. |

## References

[^1]: Faraday repository, created 2009-12-10. https://github.com/lostisland/faraday
[^2]: Faraday documentation, "Upgrading to Faraday 2.0" — adapters and middleware (retry, multipart, and non-default adapters) moved to independent gems. https://lostisland.github.io/faraday/#/upgrading/2.0

## Tags

ruby, http-client, api-client, middleware, networking, rack, adapter-pattern, rest, gem, library
