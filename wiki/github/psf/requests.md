# psf/requests

> The synchronous HTTP client most Python code reaches for by default — a readable façade over urllib3.

[GitHub repo](https://github.com/psf/requests) ·
[Official website](https://requests.readthedocs.io/en/latest/) ·
[License: Apache-2.0](https://github.com/psf/requests/blob/main/LICENSE)

## Overview

Requests is a synchronous HTTP/1.1 client library for Python, created by
Kenneth Reitz in 2011 under the "HTTP for Humans" banner[^1]. Its thesis was
that Python's standard-library `urllib`/`urllib2` were correct but painful, and
that an ergonomic wrapper — `requests.get(url)` returning a rich `Response` —
would remove enough friction to become the default. It succeeded: it is one of
the most-downloaded packages on PyPI (on the order of hundreds of millions of
downloads per week) and is a transitive dependency of a large fraction of the
Python ecosystem[^2].

The library is now a Python Software Foundation project, maintained by a small
team after Reitz stepped back, and is effectively in **feature-stable /
maintenance mode**: security fixes, compatibility with new Python and urllib3
releases, and bug fixes land, but the API surface is deliberately frozen. This
is a feature, not neglect — millions of scripts depend on its exact behavior.
The flip side is the defining tension of Requests in 2026: it is synchronous
and HTTP/1.1-only by design, at a point when the ecosystem has moved toward
async I/O and HTTP/2. The project has no plans to change that, which is why the
async-and-HTTP/2 successor conversation happens in httpx, niquests, and aiohttp
rather than here.

## Getting Started

```console
$ python -m pip install requests
```

```python
import requests

# Prefer a Session: it reuses the underlying TCP connection pool
# and persists cookies across calls.
with requests.Session() as s:
    r = s.get(
        "https://httpbin.org/get",
        params={"q": "python"},
        timeout=10,            # ALWAYS set this — see Production Notes
    )
    r.raise_for_status()       # turn 4xx/5xx into an exception
    data = r.json()

    s.post("https://httpbin.org/post", json={"hello": "world"})
```

## Architecture / How It Works

Requests is a thin, opinionated layer. The actual connection pooling, TLS,
retries, and low-level HTTP live in **urllib3**, which Requests vendors/depends
on and drives through an adapter[^3]. Understanding this split explains most of
its behavior:

- **`Session`** holds configuration (headers, auth, cookies, proxies) and a map
  of `HTTPAdapter`s mounted per URL prefix (`http://`, `https://`). Each adapter
  wraps a urllib3 `PoolManager`. A bare `requests.get()` quietly creates a
  throwaway `Session` for that one call — convenient, but it means no
  connection reuse and a new pool per call.
- **`HTTPAdapter`** is the extension seam. Connection pool sizing
  (`pool_connections`, `pool_maxsize`), retry policy (a urllib3 `Retry` object),
  and TLS context are configured by mounting a custom adapter.
- **`Response`** is eager by default: the body is fully read into `r.content`
  unless you pass `stream=True` and iterate with `iter_content()` /
  `iter_lines()`. `r.text` decodes `r.content` using an encoding guessed from
  headers (and historically `chardet`, now `charset_normalizer`)[^4].
- **Redirects, cookies, and auth** are resolved in Requests itself, above
  urllib3, which is why redirect and cookie semantics are Requests' own.

There is no event loop and no concurrency primitive inside the library. Parallelism
is the caller's problem: threads (a `Session` is not guaranteed thread-safe for
concurrent mutation, though separate threads sharing a pool is common),
`concurrent.futures`, or multiprocessing.

## Production Notes

- **No default timeout.** This is the single most common production footgun. A
  request with no `timeout=` can hang indefinitely if the server never
  responds, blocking the calling thread forever. Always pass an explicit
  `timeout` (a `(connect, read)` tuple gives the most control). Nothing in the
  library will save you here.
- **Use a `Session` for anything beyond one call.** Per-call `requests.get()`
  opens and tears down connections and defeats keep-alive; under load this shows
  up as socket exhaustion and TLS-handshake overhead. A shared `Session` reuses
  the pool.
- **Retries are opt-in and off by default.** To get backoff/retry you mount an
  `HTTPAdapter(max_retries=urllib3.util.Retry(...))`. Naïve `while` retry loops
  around `requests.get` miss connection-level failures urllib3 handles better.
- **`stream=True` responses must be consumed or closed.** Forgetting to read or
  close a streamed response leaks a connection back into the pool in a bad
  state. Use the context-manager form or call `r.close()`.
- **TLS verification is on by default (good), and disabling it is loud but
  ignored.** `verify=False` disables certificate checking and emits an
  `InsecureRequestWarning`; people suppress the warning instead of fixing the
  trust store. On corporate networks, point `verify=` at a CA bundle or set
  `REQUESTS_CA_BUNDLE`.
- **Performance ceiling is synchronous I/O.** For high-concurrency fan-out
  (hundreds of simultaneous requests), a thread-per-request model with Requests
  is heavier than an async client. This is an architectural limit, not a tuning
  problem.
- **`.json()` raises on non-JSON bodies.** It does not check `Content-Type`;
  guard with `raise_for_status()` and be ready for `requests.JSONDecodeError`.

## When to Use / When Not

**Use when:**
- You want the most widely-understood, best-documented HTTP client in Python.
- Your workload is synchronous scripts, CLI tools, glue code, or per-request
  handlers where blocking I/O is fine.
- You value stability and a frozen API over new protocol features.

**Avoid when:**
- You need async/`await`, HTTP/2, or HTTP/3 — reach for httpx, aiohttp, or niquests.
- You are doing high-concurrency fan-out where thread overhead dominates.
- You want connection-level control that is cleaner to express against urllib3
  directly.

## Alternatives

- encode/httpx — near-identical API plus async support and HTTP/2; the usual
  recommendation when you need `await` or want a modern successor.
- urllib3/urllib3 — the layer Requests sits on; use it directly when you want
  explicit pool/retry control without the Requests façade.
- aio-libs/aiohttp — asyncio-native client and server; use when your whole app
  is already async.
- jawah/niquests — drop-in Requests fork adding HTTP/2, HTTP/3, and async; use
  when you want Requests' API but modern transport.
- lexiforest/curl_cffi — libcurl-backed client with browser TLS/JA3
  impersonation; use when you must evade TLS fingerprinting.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2011 | Initial release by Kenneth Reitz; "HTTP for Humans"[^1]. |
| 1.0.0 | 2012-12 | First stable API; sessions, adapters formalized. |
| 2.0.0 | 2013-09 | Major cleanup; long-lived 2.x line begins. |
| — | 2019 | Stewardship moves to the Python Software Foundation. |
| 2.32.x | 2024–2025 | Security and compatibility fixes; charset_normalizer, Python 3.8+ then 3.10+ baselines[^4]. |

## References

[^1]: Kenneth Reitz, "Requests: HTTP for Humans" — project origin and philosophy. https://requests.readthedocs.io/en/latest/
[^2]: PyPI project page and dependents graph. https://pypi.org/project/requests/ · https://github.com/psf/requests/network/dependents
[^3]: Requests advanced usage — Transport Adapters and Sessions. https://requests.readthedocs.io/en/latest/user/advanced/
[^4]: Requests changelog / release history. https://github.com/psf/requests/blob/main/HISTORY.md

## Tags

python, http, http-client, rest, api-client, networking, urllib3, synchronous, library, psf
