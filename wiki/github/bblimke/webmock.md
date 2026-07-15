# bblimke/webmock

> HTTP request stubbing and expectation-setting for Ruby test suites, working at the HTTP-client-adapter level so tests survive swapping one HTTP library for another.

[GitHub repo](https://github.com/bblimke/webmock) ·
[License: MIT](https://github.com/bblimke/webmock/blob/master/LICENSE)

## Overview

WebMock is a Ruby library for stubbing outgoing HTTP requests and asserting
that expected requests were (or were not) made. It is one of the oldest and
most widely depended-on testing gems in the Ruby ecosystem, with the repository
dating to 2009[^1], and remains a de facto standard alongside VCR (which builds
on top of it) for isolating test suites from the network.

The defining design decision is that WebMock hooks in *below* the HTTP client
library, not above it. Rather than mocking `Net::HTTP` or `RestClient` method
calls directly, it patches the adapter layer of each supported client so one
stub matches a request regardless of which library issued it. The upside: you
can change HTTP libraries without rewriting stubs. The cost: WebMock carries a
bespoke adapter for every client it supports and must track their internals —
and clients it has no adapter for are not intercepted at all.

The other tension worth knowing up front: by default WebMock disables all real
network connections in the test environment. That is the point, but it means an
un-stubbed request raises rather than silently passing through, and
adapter-level interception can subtly alter client behavior — most notably
`Net::HTTP.start` connection semantics (see Production Notes).

## Getting Started

```bash
gem install webmock
```

```ruby
# spec/spec_helper.rb — wire WebMock into RSpec (disables net by default)
require "webmock/rspec"
```

```ruby
# stub a request and assert it happened
stub_request(:post, "https://api.example.com/charges").
  with(body: { amount: 100 }, headers: { "Authorization" => "Bearer t0ken" }).
  to_return(status: 200, body: '{"id":"ch_1"}',
            headers: { "Content-Type" => "application/json" })

MyClient.charge(100)   # any supported HTTP library

expect(WebMock).to have_requested(:post, "https://api.example.com/charges").
  with(body: { amount: 100 }).once
```

Framework entry points are `webmock/rspec`, `webmock/minitest`,
`webmock/test_unit`, and `webmock/cucumber`; outside a framework, `require
"webmock"`, `include WebMock::API`, and call `WebMock.enable!`.

## Architecture / How It Works

WebMock's core is a registry of stubs and a request-matching engine. A
`stub_request` builds a `RequestPattern` (method, URI, headers, body, query)
and pairs it with one or more `ResponsesSequence` entries. Every intercepted
request is normalized into a `RequestSignature` and compared against registered
patterns; the first match wins, and if none matches WebMock raises a
`NetConnectNotAllowedError` whose message includes a copy-pasteable stub
suggestion.

Interception is per-client. WebMock ships a `HttpLibAdapter` for each supported
library — Net::HTTP (and everything layered on it: HTTParty, RestClient,
Faraday's default adapter), Curb, Excon, HTTPClient, http.rb, httpx, Typhoeus
(Hydra), Patron, EM-HTTP-Request, Async::HTTP, and Manticore[^2]. Each adapter
monkey-patches the client's request path to route through WebMock's stub
registry when enabled and to restore original behavior when disabled. This is
why "supported libraries" is a finite, enumerated list: a client with no
adapter is invisible to WebMock and its requests go to the real network.

Matching is deliberately lenient about representation. URIs compare in both
encoded and decoded forms; query strings and form/JSON/XML bodies compare
structurally rather than byte-for-byte (so `hash_including`/`hash_excluding`
partial matches work); headers normalize for case and representation. URIs can
also match by regexp, a `->(uri)` lambda, or an `Addressable::Template` (RFC
6570). Responses can be static, sequenced (each call returns the next, last
repeats forever), evaluated from a block/lambda per request, raw `curl -is`
dumps, or delegated to a Rack app via `to_rack`.

## Production Notes

This is a test-only dependency, so "production" here means CI reliability and
test-suite correctness. The real footguns:

- **`Net::HTTP.start` semantics change.** WebMock defers connecting until a
  request is actually made, so opening a connection with `Net::HTTP.start` and
  making no request does nothing. The README states plainly that this "breaks
  the Net::HTTP behaviour by default." If code under test relies on connect
  timing, pass `net_http_connect_on_start: true` to
  `allow_net_connect!`/`disable_net_connect!`.
- **Un-stubbed requests raise, they do not pass through.** Once WebMock is
  required in the test env, `disable_net_connect!` is in effect. A newly added
  code path that makes an un-stubbed call fails the test rather than hitting the
  network — usually desirable, occasionally surprising in integration specs.
  Use `disable_net_connect!(allow_localhost: true)` or `allow:` (String,
  Regexp, `#call`-able, or Array thereof) to whitelist, e.g. a Selenium or test
  server on localhost.
- **Auth-matching break since 2.0.0.** WebMock does not treat credentials in an
  `Authorization` header as equivalent to credentials in a URL userinfo. A stub
  written as `stub_request(:get, "user:pass@example.com")` will not match a
  request that sends basic auth via the header, and vice versa. This bit many
  suites during the 1.x→2.x upgrade[^3].
- **Global mutable state leaks between tests.** Stubs and the net-connect flag
  are process-global. Framework integrations reset state per example
  (`WebMock.reset!`), but manual `WebMock.enable!` usage must reset explicitly
  or stubs leak and cause order-dependent failures.
- **Adapter coverage lags client releases.** A new major version of an HTTP
  client (or a client with no adapter) can silently escape stubbing until
  WebMock ships an updated adapter. Pin and verify after upgrading HTTP libs.
- **VCR coupling.** Most VCR setups use WebMock as the underlying request hook,
  so a WebMock major upgrade can affect cassette replay.

## When to Use / When Not

**Use when:**
- You need deterministic, offline unit/integration tests for code that makes
  HTTP calls, and you want stubs that survive changing HTTP client libraries.
- You want to assert on *outgoing* request shape (method, URI, body, headers,
  query) as part of a test's contract.
- You use VCR and need its underlying stubbing engine.

**Avoid / look elsewhere when:**
- You want to record and replay real interactions rather than hand-write stubs —
  use VCR (on top of WebMock) instead of raw WebMock.
- Your HTTP client has no WebMock adapter; its traffic won't be intercepted.
- You need to fake a whole server for manual/local runs or cross-language tests
  — a real stub server (WireMock, a local Sinatra/Rack app) fits better than
  in-process interception.
- You are mocking at the object/method boundary rather than the HTTP boundary —
  plain RSpec mocks may be simpler and faster.

## Alternatives

- vcr/vcr — record real HTTP interactions to cassettes and replay them; use it
  when hand-writing stubs is tedious and you want fixtures from real responses
  (it uses WebMock underneath).
- chrisk/fakeweb — the older, now largely unmaintained predecessor; only
  relevant for legacy suites still on it.
- bblimke/webmock is Ruby-specific — for a language-agnostic HTTP mock server
  use wiremock/wiremock (JVM) run as an external process.
- thoughtbot/capybara-style stub servers or a local Sinatra app — use when you
  need a real listening socket rather than in-process patching.
- Faraday's built-in `Faraday::Adapter::Test` — use for Faraday-only stacks
  where you want stubbing without patching the global HTTP layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2009-11 | First release; Net::HTTP stubbing with adapter-level interception[^1]. |
| 1.x | 2010–2016 | Long-lived line; broad HTTP-client adapter coverage added. |
| 2.0.0 | 2016 | Breaking changes incl. Authorization-header vs URL-userinfo credential matching no longer equivalent[^3]. |
| 3.x | 2017–present | Current major line; ongoing adapter and Ruby-version support (through MRI 4.0 and JRuby)[^2]. |

Actively maintained: last push 2026-03-18, ~4.1k stars and ~576 forks, MIT
licensed, with a ~180 open-issue count typical of a long-lived library tracking
many downstream HTTP clients. Intermediate release dates were not verified
against the changelog and are given as year ranges.

## References

[^1]: WebMock repository, created 2009-11-06. https://github.com/bblimke/webmock
[^2]: WebMock README, "Supported HTTP libraries" and "Supported Ruby Interpreters". https://github.com/bblimke/webmock/blob/master/README.md
[^3]: WebMock README note: since version 2.0.0, credentials in the Authorization header and in URL userinfo are matched separately. https://github.com/bblimke/webmock/blob/master/README.md

## Tags

ruby, testing, http, mocking, stubbing, test-double, net-http, rspec, minitest, integration-testing, vcr
