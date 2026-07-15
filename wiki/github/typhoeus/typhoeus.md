# typhoeus/typhoeus

> A Ruby HTTP client that wraps libcurl and runs many requests in parallel through a single-threaded curl_multi event loop.

[GitHub repo](https://github.com/typhoeus/typhoeus) ·
[API docs](https://rubydoc.info/github/typhoeus/typhoeus) ·
[License: MIT](https://github.com/typhoeus/typhoeus/blob/master/LICENSE)

## Overview

Typhoeus is a Ruby HTTP client, first released by Paul Dix in 2009, whose
distinguishing feature is parallel request execution built on libcurl's
`curl_multi` interface rather than on Ruby threads[^1]. Its `Hydra` class
queues an arbitrary number of requests and drives them concurrently inside a
single OS thread, which is why it has long been the default choice when a Ruby
process needs to fan out hundreds of outbound calls (crawlers, API aggregators,
warmup jobs) without paying for a thread per connection.

Since the 0.5 rewrite (2012) Typhoeus no longer talks to libcurl through its own
C extension; it delegates the FFI binding to a sibling gem, Ethon[^2]. Typhoeus
is the ergonomic layer — requests, responses, callbacks, stubbing, caching — and
Ethon is the thin `Ffi::Easy`/`Ffi::Multi` mapping over the C library. This split
is the single most important thing to understand about the project: most option
names, edge-case behavior, and version-dependent quirks originate in libcurl and
are simply passed through.

The defining tradeoff is dependency weight for throughput. You get libcurl's
mature networking (HTTP/2, SSL backends, proxy support, connection reuse) and
non-blocking parallelism, but you take on a native dependency whose behavior
varies with the system libcurl build, and an API that fails silently by design on
timeouts. It is at home in batch/parallel workloads and awkward for one-off calls
where a pure-Ruby client would do.

## Getting Started

```bash
gem install typhoeus        # or: bundle add typhoeus
```

```ruby
require "typhoeus"

# Single blocking request
res = Typhoeus.get("https://example.com", followlocation: true)
res.code        #=> 200
res.total_time  #=> 0.234

# Parallel requests via Hydra
hydra = Typhoeus::Hydra.new(max_concurrency: 20)
requests = 10.times.map do
  req = Typhoeus::Request.new("https://example.com", followlocation: true)
  hydra.queue(req)
  req
end
hydra.run  # blocks until all queued requests finish
bodies = requests.map { |r| r.response.body }
```

## Architecture / How It Works

The public surface is three classes: `Request`, `Response`, and `Hydra`.
A `Request` is a description plus a set of callbacks (`on_complete`,
`on_headers`, `on_body`); a `Response` is populated after the request runs;
`Hydra` is the scheduler.

Under the hood every `Request` is compiled into an Ethon `Easy` handle — a
wrapped libcurl easy handle whose options map directly to `CURLOPT_*` constants.
`Hydra` owns a libcurl multi handle and a pool of easy handles. Calling
`hydra.run` enters libcurl's multi loop: it starts up to `max_concurrency`
transfers, waits on their sockets, fires each request's `on_complete` as it
finishes, and drains the queue until empty. This is cooperative concurrency on
one thread, not parallelism — there are no Ruby threads and no GIL contention,
but any CPU-bound work inside a callback blocks the whole loop.

Because the queue is dynamic, an `on_complete` callback can queue further
requests mid-run (build request C from the body of request A), and Hydra picks
them up immediately. This makes dependent request graphs expressible without
manual synchronization.

Several features are implemented as global, process-wide interception points on
`Typhoeus::Config` and the expectation registry:

- **Stubbing** — `Typhoeus.stub(url_or_regex).and_return(response)` registers an
  expectation consulted before any real transfer; used to keep tests off the
  network.
- **Caching** — assign any object responding to `get(request)`/`set(request, response)`
  to `Typhoeus::Config.cache`. Built-in adapters ship for Dalli, Redis, and Rails
  cache, each accepting a `default_ttl`.
- **Memoization** — `Typhoeus::Config.memoize` deduplicates identical requests
  within a single `hydra.run`; all duplicate callbacks still fire.

Typhoeus also registers itself as a Faraday adapter (`builder.adapter :typhoeus`),
which is how many applications consume it — Faraday's `in_parallel` block maps
onto a Hydra run.

## Production Notes

**Native dependency.** Typhoeus needs libcurl at runtime, and behavior tracks the
system libcurl version and how it was compiled. SSL support, HTTP/2, and
asynchronous DNS resolution are all libcurl build flags. Sub-second DNS timeouts
require an async resolver in the libcurl build, and `timeout_ms`/`connecttimeout_ms`
millisecond precision is unavailable on some platforms unless `nosignal` is set.
Two machines with different libcurl builds can behave differently with identical code.

**Timeouts fail silently.** No exception is raised on a timeout — this is
deliberate. You must inspect `response.timed_out?`, and `response.code == 0`
signals a transport-level failure (with detail in `response.return_message`).
Code that assumes a raise on failure will treat timeouts as empty successes; this
is the most common Typhoeus footgun in production.

**Global mutable state.** Stubs, cache, memoize, verbose, and `user_agent` live on
`Typhoeus::Config` and the global expectation registry — process-wide, not
per-request. Stubs in particular persist across examples; the standard remedy is
`Typhoeus::Expectation.clear` in an RSpec `before(:each)`. Forgetting it produces
tests that pass or fail depending on order.

**Concurrency model.** `max_concurrency` defaults to 200; pushing it high can
overwhelm remote hosts and the local FD limit. A `Hydra` is not designed to be
shared across threads — treat one Hydra as belonging to one thread and one `run`
at a time. Heavy per-response processing in `on_complete` serializes against the
event loop, so offload CPU work.

**Streaming.** Setting an `on_body` callback switches the response to streaming
mode and stops buffering — `response.body` will be `""`. To abort a stream mid-flight,
return `:abort` from the `on_body` block; interrupting with `return`/`raise` can leak
memory inside the transfer.

## When to Use / When Not

**Use when:**
- You need to issue many concurrent outbound requests from one process without a
  thread-per-request model.
- You want libcurl's networking maturity (proxies, cookies, SSL knobs, HTTP/2,
  connection reuse) exposed as Ruby options.
- You already use Faraday and want a high-throughput parallel adapter.

**Avoid when:**
- You make mostly single, sequential requests — a pure-Ruby client avoids the
  native dependency and the silent-timeout semantics.
- Installing libcurl development headers in your build/deploy environment is
  painful or unsupported.
- Your concurrency is fiber- or thread-based (e.g. the `async` ecosystem);
  Typhoeus's own event loop does not compose with those schedulers.

## Alternatives

- lostisland/faraday — HTTP abstraction with swappable adapters; use it when you want a stable API and Typhoeus as just one interchangeable backend.
- httprb/http — pure-Ruby client with a clean chainable API; use when you want no native dependency and don't need libcurl-driven parallelism.
- excon — pure-Ruby client with persistent connections (the fog/AWS lineage); use for simple pooled request/response without a curl_multi loop.
- socketry/async-http — fiber-based HTTP for high concurrency on modern Ruby; use when you're already building on the async reactor.
- typhoeus/ethon — the libcurl FFI binding underneath Typhoeus; use directly when you want raw easy/multi handles without Typhoeus's request/stub/cache layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2009 | Initial release by Paul Dix; libcurl via a bundled C extension[^1]. |
| 0.5 | 2012 | Major rewrite; C extension dropped in favor of the Ethon FFI binding[^2]. |
| 1.0 | 2016 | First stable major; API stabilized under semantic versioning. |
| 1.4.x | 2020– | Current line; maintenance, libcurl/Ethon compatibility, Ruby version support. |

## References

[^1]: Typhoeus project history and authorship (Paul Dix 2009; David Balatero; Hans Hasselberg), README license section. https://github.com/typhoeus/typhoeus
[^2]: Ethon — libcurl FFI binding used by Typhoeus since the 0.5 rewrite. https://github.com/typhoeus/ethon
[^3]: libcurl multi interface (the concurrency primitive Hydra drives). https://curl.se/libcurl/c/libcurl-multi.html

## Tags

ruby, http-client, libcurl, parallel-requests, networking, curl, faraday-adapter, concurrency, rubygem
