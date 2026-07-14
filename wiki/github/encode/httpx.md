# encode/httpx

> A requests-style HTTP client for Python with first-class async, HTTP/2, and strict-by-default timeouts — still versioned 0.x after seven years.

[GitHub repo](https://github.com/encode/httpx) ·
[Official website](https://www.python-httpx.org/) ·
[License: BSD-3-Clause](https://github.com/encode/httpx/blob/master/LICENSE.md)

## Overview

HTTPX is a full-featured HTTP client for Python 3, first released in 2019 by the Encode org (Tom Christie, the author of Django REST Framework, Starlette, and Uvicorn)[^1]. Its pitch is a `requests`-compatible API that also offers a native async interface, HTTP/2, and an integrated command-line client. It has become the default client for the async Python ecosystem — FastAPI's and Starlette's `TestClient` are built on it, and most ASGI stacks reach for it over `aiohttp` when they want request/response ergonomics rather than a full client+server framework.

The defining tension is that HTTPX aims to feel like `requests` while deliberately breaking with it where `requests`' defaults are unsafe. Timeouts are enforced everywhere with a 5-second default (requests has no default timeout); redirects are *not* followed unless you ask; and the ergonomic top-level functions (`httpx.get`, etc.) are explicitly discouraged for anything beyond one-off calls because they spin up and tear down a connection pool per call. The result is a library that reads as familiar but punishes copy-pasting `requests` code without reading the compatibility notes[^2].

A second point worth stating plainly: after seven years and heavy production use, HTTPX has never shipped a 1.0. The API is stable in practice, but the maintainers reserve the right to make breaking changes across 0.x minors, and they do.

## Getting Started

```shell
pip install httpx
# optional extras:
pip install 'httpx[http2]'   # HTTP/2 via h2
pip install 'httpx[cli]'     # the `httpx` command-line client
```

```python
import httpx

# One-off call — convenient, but opens/closes a pool each time.
r = httpx.get("https://www.example.org/", timeout=10.0)
print(r.status_code, r.headers["content-type"])

# Reuse a Client for connection pooling (the recommended pattern).
with httpx.Client(base_url="https://api.example.com") as client:
    r = client.get("/users", params={"page": 1})
    r.raise_for_status()

# Async — same surface, awaitable.
import asyncio

async def main():
    async with httpx.AsyncClient() as client:
        r = await client.get("https://www.example.org/")
        print(r.text[:80])

asyncio.run(main())
```

## Architecture / How It Works

HTTPX is a thin, opinionated layer over a stack of Encode-maintained pieces:

- **httpcore** — the actual transport (connection pooling, keep-alive, proxy handling, the sync/async network I/O). HTTPX owns the `requests`-style API surface; httpcore owns the sockets[^3].
- **h11** for HTTP/1.1 and **h2** (optional) for HTTP/2 — both are sans-I/O protocol state machines, so the same parsing code runs under sync and async.
- **anyio** underneath the async path, which is why `AsyncClient` runs on both asyncio and Trio; `sniffio` autodetects the running loop.
- **certifi** (CA bundle), **idna** (internationalized domains).

The sync and async clients are two implementations of one design, not a shim: `Client` and `AsyncClient` share request-building, redirect, auth, and cookie logic, differing only at the transport call. Both are built around a **Transport** abstraction. That abstraction is the single most useful architectural choice in the library — because `WSGITransport` and `ASGITransport` let you route requests straight into a Python web app in-process with no socket, which is exactly how FastAPI/Starlette test clients work[^4].

Redirect handling, auth flows (Basic, Digest, and custom `Auth` classes that can issue follow-up requests), cookie persistence, and `.netrc` support all live in HTTPX itself. HTTP/2 is negotiated via ALPN only when the `h2` extra is installed; there is no HTTP/3 support.

## Production Notes

- **Don't use the top-level API in hot paths.** `httpx.get(...)` constructs a fresh `Client`, opens a pool, and closes it every call. Under load this destroys keep-alive and TLS session reuse. Create one long-lived `Client`/`AsyncClient` and share it.
- **Clients are not free to leak.** A `Client` that is never `.close()`d (or used outside a `with` block) leaks connections and emits resource warnings. In async code, an `AsyncClient` must be closed on the same loop it was opened on — closing across loops or after the loop is gone is a recurring source of confusing errors.
- **Pool limits are the throughput knob.** Defaults cap total and per-host connections (`httpx.Limits`). High-concurrency async workloads that don't raise `max_connections` will silently queue behind the pool rather than error — latency spikes that look like the server being slow are often the local pool.
- **Timeouts are strict and granular.** The 5s default applies to connect, read, write, and pool-acquire independently. Long-polling, streaming, or slow upstreams need an explicit `timeout=` (or `None`) or they raise `ReadTimeout` mid-response.
- **The two most common migration surprises from requests:** redirects are off by default (pass `follow_redirects=True`), and there is no implicit retry — HTTPX ships no retry logic beyond httpcore's connection-level retries, so application retries are your job (or a `RetryTransport`).
- **Sync client inside an event loop** works but blocks the loop; use `AsyncClient` in async code rather than calling the sync client from a coroutine.
- **HTTP/2 is opt-in and per-client** (`httpx.Client(http2=True)` + the extra). Forgetting the extra silently falls back to HTTP/1.1.

## When to Use / When Not

**Use when:**
- You need one client that speaks both sync and async with the same API.
- You want `requests`-like ergonomics in an asyncio/Trio codebase.
- You're testing an ASGI/WSGI app and want in-process requests via a transport.
- You want HTTP/2 or strict timeout behavior without hand-rolling it.

**Avoid when:**
- You want a rock-stable, 1.0-guaranteed API with the largest possible ecosystem of adapters — `requests` is more conservative.
- You need an HTTP *server* too, or a battle-tested asyncio-native stack — `aiohttp` bundles both.
- You need HTTP/3, which HTTPX does not implement.
- You need built-in retry/backoff out of the box — you'll add it yourself either way.

## Alternatives

- psf/requests — the sync-only classic; use it when you don't need async or HTTP/2 and want maximum ecosystem and adapter compatibility.
- aio-libs/aiohttp — async-first client and server; use it when you want a mature asyncio-native stack or need to run an HTTP server in the same library.
- encode/httpcore — the transport layer beneath HTTPX; use it directly when you want low-level connection/pool control without the requests-style surface.
- urllib3/urllib3 — the low-level building block under requests; use it when you need fine-grained control over pooling and retries.
- jawah/niquests — a `requests` drop-in fork adding HTTP/2, HTTP/3, and async; use it when you want the requests API with modern protocols.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.7.x | 2019 | First public releases under `encode/httpx`; sync + async client. |
| 0.20.0 | 2021-09 | `follow_redirects` replaces `allow_redirects`; redirects no longer followed by default[^5]. |
| 0.23.x | 2022 | Consolidation of the transport/proxy API; deprecations begin. |
| 0.24.0 | 2023 | Dropped several deprecated shortcuts; tightened proxy configuration. |
| 0.27.x | 2024 | Deprecated the `app=` and `proxies=` client shortcuts in favor of explicit transports/mounts. |
| 0.28.x | 2024 | Removed the deprecated `app=` / `proxies=` shortcuts. |

*(Still pre-1.0 as of 2026; minor versions may include breaking changes.)*

## References

[^1]: Encode — organization behind HTTPX, Starlette, Uvicorn, and httpcore. https://www.encode.io/
[^2]: HTTPX docs, "Requests Compatibility" — enumerates deliberate differences from `requests` (redirects, timeouts, top-level API). https://www.python-httpx.org/compatibility/
[^3]: encode/httpcore — the underlying transport implementation used by HTTPX. https://github.com/encode/httpcore
[^4]: HTTPX docs, "Transports" — WSGI/ASGI transports for in-process requests. https://www.python-httpx.org/advanced/transports/
[^5]: HTTPX changelog. https://github.com/encode/httpx/blob/master/CHANGELOG.md

## Tags

python, http-client, async, asyncio, trio, http2, requests-compatible, networking, rest, cli
