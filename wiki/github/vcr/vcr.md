# vcr/vcr

> Record your Ruby test suite's real HTTP interactions once, then replay them from disk for fast, deterministic, offline tests.

[GitHub repo](https://github.com/vcr/vcr) ·
[Documentation](https://benoittgt.github.io/vcr) ·
[License: Hippocratic License 2.1](https://github.com/vcr/vcr/blob/master/LICENSE) (not OSI-approved)

## Overview

VCR is the canonical HTTP-replay library for Ruby. You wrap a block of test code in `VCR.use_cassette`; the first run records every HTTP request and response into a YAML (or JSON) "cassette" file on disk, and every subsequent run replays those recorded responses instead of hitting the network. The result is a test that is fast (no real I/O), deterministic (passes offline and when the upstream API is down), and accurate (it replays the actual bytes a real server returned)[^1]. It was created by Myron Marston around 2010 and is now maintained by a small community team under the `vcr/` org, with Benoit Tigeot hosting the current docs.

VCR does not intercept HTTP itself — it is a coordination layer that hooks into an existing stubbing library (WebMock is the usual choice; Typhoeus, Faraday, and Excon are also supported). That indirection is its defining architectural trait and the source of most version-compatibility friction: a VCR upgrade, a WebMock upgrade, and your HTTP client library all have to agree.

The defining tension is **recorded-fixture staleness versus test speed**. A cassette is a snapshot frozen at record time. It never notices that the upstream API changed its response shape, deprecated a field, or started returning a different error format — the test keeps passing against a fiction. VCR buys determinism at the cost of drift, and teams that treat cassettes as write-once fixtures eventually ship code against an API that no longer exists in that form.

## Getting Started

```ruby
# Gemfile
gem "vcr"
gem "webmock"   # VCR needs a hook library; WebMock is the common default
```

```ruby
# spec_helper.rb
require "vcr"

VCR.configure do |config|
  config.cassette_library_dir = "spec/cassettes"
  config.hook_into :webmock
  config.filter_sensitive_data("<API_KEY>") { ENV["API_KEY"] }
end
```

```ruby
# a test — first run records to spec/cassettes/iana.yml, later runs replay it
VCR.use_cassette("iana") do
  response = Net::HTTP.get_response(URI("http://www.iana.org/domains/reserved"))
  expect(response.body).to match(/Example domains/)
end
```

With RSpec metadata you can drop the explicit block: `it "...", :vcr do ... end` records a cassette named after the example.

## Architecture / How It Works

VCR sits between your test and a stubbing library:

1. **Hooks** — `hook_into :webmock` (or `:typhoeus`, `:faraday`, `:excon`) registers VCR as a request handler inside that library. Any HTTP client built on the hooked layer (Net::HTTP, HTTParty, Rest Client, Mechanize, Faraday adapters) is transparently intercepted[^2].
2. **Cassettes** — a cassette is a serialized list of `http_interactions`, each an HTTP request paired with its response (status, headers, body). They live as plain YAML files you commit to the repo and can inspect or hand-edit.
3. **Record modes** — `:once` (default: record if the cassette is absent, replay if present, never mix), `:none` (replay only; error on any unrecorded request), `:new_episodes` (replay known requests, record new ones), and `:all` (always re-record). Choosing the wrong mode is the most common source of confusing behavior.
4. **Request matching** — by default VCR matches a replay to a recording on **method + URI**. You override with `match_requests_on: [:method, :uri, :body, :headers]` or a custom matcher lambda. Matching is what breaks when requests carry timestamps, nonces, or randomized bodies.
5. **Serializers** — YAML by default, JSON built in, and a pluggable interface for custom formats. Responses can be templated with ERB for dynamic values.

By default VCR **disables all HTTP requests that no cassette explicitly allows** — an unmatched request raises rather than silently escaping to the network. `allow_http_connections_when_no_cassette` and `ignore_hosts` / `ignore_localhost` carve out exceptions (commonly for a Capybara/Selenium driver talking to a local app server).

## Production Notes

**License is the first thing to check with legal.** Since v5.0.0 (2019) VCR is distributed under the **Hippocratic License 2.1**, an ethical-source license with human-rights use restrictions[^3]. It is **not OSI-approved**, which is why GitHub's detector reports the repo as `NOASSERTION` rather than a recognized SPDX id. Some corporate legal and compliance policies reject non-OSI licenses outright. Versions up to and including the 4.x line were MIT; teams with strict license allowlists sometimes pin to an older MIT release or vendor a fork for this reason.

**Cassettes leak secrets by default.** A recorded response captures real `Authorization` headers, `Set-Cookie` values, tokens, and PII, and VCR will happily commit them to git. `filter_sensitive_data` must be configured **before** the recording is made — it does not retroactively scrub cassettes already on disk. Auditing existing cassettes for leaked credentials is a routine part of adopting VCR on a real codebase.

**Staleness is silent.** A cassette recorded in `:once` mode never expires on its own. The upstream API can change and your suite stays green against the frozen snapshot. `re_record_interval` forces periodic re-recording, but it requires the ability to reach the live API in CI (and to re-inject filtered secrets), which many pipelines cannot do.

**Thread and parallel-test safety is limited.** VCR manages the current cassette in per-thread state, and concurrent tests writing the same cassette file can corrupt or race it. Parallel runners (`parallel_tests`, `flatware`) generally need per-process cassette isolation; sharing one cassette across parallel workers is a known footgun.

**Matching brittleness.** The default method+URI match ignores the body, so two different POSTs to the same URL collide onto one recording. Turn on `:body` matching and any timestamp or nonce in the payload now breaks replay. Most real setups end up writing a custom matcher.

**Version coupling.** VCR, WebMock, and the underlying HTTP library must be mutually compatible; a WebMock major bump can require a VCR upgrade. Binary or gzip-compressed response bodies also need care in the YAML serializer to avoid encoding corruption.

## When to Use / When Not

**Use when:**
- You have integration tests against third-party HTTP APIs and want them fast, offline, and deterministic in CI.
- You want to avoid rate limits, flaky upstreams, or paid API calls on every test run.
- The recorded response shape is stable enough that periodic re-recording is realistic.

**Avoid when:**
- You are verifying live integration correctness — a replayed snapshot cannot catch a breaking upstream change (use contract testing such as Pact, or scheduled live smoke tests).
- Requests are inherently non-deterministic (streaming, websockets, per-request signed bodies) and matching becomes a maintenance sink.
- Your organization forbids non-OSI licenses and cannot use the Hippocratic License terms.
- You are not on Ruby — use one of the language ports below.

## Alternatives

- bblimke/webmock — the lower-level stubbing library VCR sits on top of; use it directly when you want to hand-write explicit request stubs instead of recording real traffic.
- kevin1024/vcrpy — the Python port of the same record/replay model; use it when your suite is Python, not Ruby.
- nock/nock — HTTP interception for Node.js; use it for JavaScript/TypeScript test suites.
- pact-foundation/pact — consumer-driven contract testing; use it when you need to verify both sides of an integration stay compatible rather than replay a frozen snapshot.
- oesmith/puffing-billy — a caching proxy for stubbing HTTP in browser-driven (JS) tests; use it when the requests originate from a real browser, not Ruby code.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | ~2010 | Initial release by Myron Marston; cassette record/replay concept[^1]. |
| 2.0 | 2012 | Rewrite; pluggable hook libraries and serializers. |
| 3.0 | 2016 | API cleanup; dropped older Ruby support. |
| 5.0 | 2019 | Relicensed from MIT to the Hippocratic License[^3]. |
| 6.0.0 | 2020-05-28 | Major release; modernized Ruby support baseline. |
| 6.1.0 | 2022-03-13 | Last version supporting Ruby >= 2.6. |
| 6.2.0 | 2023-06-26 | Maintenance and compatibility updates. |
| 6.3.1 | 2024-08-20 | Bug-fix release on the 6.3 line. |
| 6.4.0 | 2025-12-22 | Latest release; tested through MRI 4.0. |

## References

[^1]: VCR README — recording and replaying HTTP interactions. https://github.com/vcr/vcr/blob/master/README.md
[^2]: VCR documentation — supported HTTP libraries and hook_into. https://benoittgt.github.io/vcr
[^3]: Hippocratic License 2.1 — ethical-source license used by VCR since 2019 (not OSI-approved). https://firstdonoharm.dev/

## Tags

ruby, testing, http, mocking, test-fixtures, vcr, cassettes, webmock, integration-testing, deterministic-tests, rspec, ethical-source-license
