# jnunemaker/httparty

> A Ruby DSL that wraps the standard-library `Net::HTTP` in a terse, convention-driven HTTP client.

[GitHub repo](https://github.com/jnunemaker/httparty) ·
[License: MIT](https://github.com/jnunemaker/httparty/blob/main/MIT-LICENSE)

## Overview

HTTParty is one of the oldest and most widely installed HTTP client gems in the Ruby ecosystem, created by John Nunemaker in 2008[^1]. Its pitch — "makes http fun again" — is a reaction to the ceremony of using Ruby's built-in `Net::HTTP` directly: HTTParty collapses request construction, connection handling, and response parsing into a handful of class methods (`HTTParty.get`, `.post`, …) and an `include HTTParty` mixin that turns any class into a small API client.

The defining tradeoff is that HTTParty is a *convenience layer*, not a transport. It is a thin DSL over `Net::HTTP`, so it inherits that library's characteristics: synchronous/blocking I/O, no built-in connection pooling, no HTTP/2, and a per-request connection model by default. What HTTParty adds is ergonomics — automatic response parsing keyed on `Content-Type`, a chainable configuration DSL (`base_uri`, `default_options`, `headers`, `format`), and a response object that behaves like the parsed body while still exposing HTTP metadata. That ergonomics-over-control stance is why it remains popular for quick scripts and simple API wrappers, and why larger applications frequently migrate off it toward more configurable clients.

## Getting Started

```bash
gem install httparty
# or add to a Gemfile:
# gem "httparty"
```

```ruby
require "httparty"

# One-off request — response auto-parsed from Content-Type (JSON here)
response = HTTParty.get("https://api.stackexchange.com/2.2/questions?site=stackoverflow")
puts response.code            # => 200 (Integer)
puts response.parsed_response # => Hash, already decoded from JSON
puts response.headers["content-type"]

# Or fold configuration into your own client class
class StackExchange
  include HTTParty
  base_uri "api.stackexchange.com"

  def questions(site: "stackoverflow", page: 1)
    self.class.get("/2.2/questions", query: { site: site, page: page })
  end
end

StackExchange.new.questions
```

## Architecture / How It Works

HTTParty is structured as two entry points over a shared request pipeline. The **class methods** (`HTTParty.get` etc.) are the anonymous path; the **mixin** (`include HTTParty`) copies a set of class-level DSL methods (`base_uri`, `default_options`, `headers`, `basic_auth`, `format`, `parser`) onto the including class and stores their results in per-class configuration. Every call ultimately builds an `HTTParty::Request`, which resolves options, constructs a `Net::HTTP` connection, performs the exchange, and wraps the result in an `HTTParty::Response`.

Two design decisions dominate day-to-day behavior:

- **Content-Type–driven parsing.** By default the response body is parsed according to the server's `Content-Type` header: JSON via the JSON stdlib, XML via `multi_xml`, plus HTML/plain/CSV handling. `HTTParty::Response` uses `method_missing` delegation so `response["key"]` reaches into the parsed structure while `response.code`, `.headers`, and `.body` expose the raw exchange. This is what makes the library feel effortless — and also means the *server* chooses how your bytes are interpreted unless you pin `format:` explicitly.
- **Class-level mutable configuration.** The mixin's DSL mutates state on the class object itself. This is convenient for a single-purpose client but is shared, global-ish state: `default_params`, `headers`, and `base_uri` set on a class are visible to every instance and every thread using it.

The transport is entirely `Net::HTTP`. HTTParty exposes its knobs — `open_timeout`, `read_timeout`, `write_timeout`, redirect following (`follow_redirects`, `max_redirects`), TLS options, and basic/digest auth — but does not replace or abstract the connection layer the way an adapter-based client does. There is no swappable backend: you get `Net::HTTP`, or you get another gem.

## Production Notes

- **Non-2xx responses do not raise by default.** `HTTParty.get` returns a `Response` for a 404 or 500 just as it does for a 200; you must inspect `response.code` / `response.success?`. Recent versions add a `raise_on:` option (e.g. `raise_on: [404, 500]`) to opt into exceptions, but code written against older assumptions silently proceeds on error responses. This is the single most common HTTParty bug in the wild.
- **No connection reuse by default.** Each request opens and tears down a connection, so high-throughput callers pay repeated TCP/TLS handshakes. Keep-alive/persistent connections require additional setup (e.g. the `persistent_connection_adapter` gem) rather than being on by default.
- **Blocking I/O, thread-per-request scaling.** Because the transport is synchronous `Net::HTTP`, concurrency comes only from your own threads/fibers or process model. There is no built-in parallel-request facility; workloads that fan out to many endpoints are better served by a libcurl- or HTTP/2-based client.
- **Automatic parsing is a footgun surface.** Trusting `Content-Type` for parser selection means a misbehaving or hostile server can steer decoding; XML parsing in particular (via `multi_xml`) has historically been a place to be cautious about untrusted input. Set `format: :json` (or a custom `parser`) explicitly when talking to untrusted or ambiguous endpoints.
- **Shared class configuration and threads.** Mutating `headers`/`default_params` on an `include HTTParty` class at runtime affects all callers. Treat class-level DSL config as set-once-at-boot, and pass per-call options (`query:`, `headers:`, `body:`) for anything request-specific.
- **Timeouts are opt-in.** With no `timeout`/`open_timeout`/`read_timeout` set you inherit `Net::HTTP`'s defaults, which are long. Always set timeouts for anything user-facing.

## When to Use / When Not

**Use when:**
- You need a quick API client or script and value terseness over configurability.
- The workload is low-to-moderate volume and synchronous request/response is fine.
- You want zero-config response parsing and a small dependency footprint over the stdlib.
- You're wrapping a single well-behaved JSON/XML API in a class.

**Avoid when:**
- You need connection pooling, HTTP/2, or high-concurrency parallel requests.
- You want swappable adapters, composable middleware (retries, instrumentation, auth), or test stubbing baked into the client — Faraday is built for this.
- You require strict, explicit error handling as the default rather than opt-in.
- The endpoint is untrusted and you want tight control over how bodies are parsed.

## Alternatives

- lostisland/faraday — middleware-and-adapter architecture; the default choice when you need retries, instrumentation, or to swap the underlying HTTP backend.
- httprb/http (http.rb) — chainable, explicit client with no global state; use when you want HTTParty-like ergonomics without class-level configuration.
- typhoeus/typhoeus — libcurl-based; use when you need parallel/concurrent requests and connection reuse.
- excon — used by fog and other libraries; use when you want persistent connections and a small, dependency-light client.
- HoneyryderChuck/httpx — modern client with HTTP/2 and native concurrency; use when protocol features and performance matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2008-08 | Initial release; `Net::HTTP` DSL wrapper[^1]. |
| 0.10.x | 2016 | Long-lived stable line widely used across the ecosystem. |
| 0.14.x | 2017 | `raise_on` option for opting into error exceptions. |
| 0.18.x | 2019 | Continued 0.18 stabilization; Ruby version floor raised over time. |
| 0.20.0 | 2021 | First 0.20 line; ongoing maintenance and dependency updates. |
| 0.21.0 | 2021 | Maintenance release on the 0.2x series. |

## References

[^1]: John Nunemaker, httparty — project on GitHub, first released 2008. https://github.com/jnunemaker/httparty
[^2]: httparty README — install, requirements (Ruby 2.7.0+), and usage examples. https://github.com/jnunemaker/httparty/blob/main/README.md
[^3]: httparty docs directory — options, timeouts, parsing, and configuration. https://github.com/jnunemaker/httparty/tree/main/docs

## Tags

ruby, http-client, rest, api-client, net-http, networking, gem, json, xml, dsl
