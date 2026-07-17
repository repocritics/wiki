# encode/httpcore

> The minimal, sans-I/O HTTP transport that sends requests and nothing else — the layer httpx is built on.

[GitHub repo](https://github.com/encode/httpcore) ·
[Official website](https://www.encode.io/httpcore/) ·
[License: BSD-3-Clause](https://github.com/encode/httpcore/blob/master/LICENSE.md)

## Overview

httpcore is a low-level HTTP client library from Encode (Tom Christie's org, also home to httpx, Starlette, and Uvicorn)[^1]. Its stated scope is deliberately narrow: send an HTTP request over a connection and return the response. It does not do redirects, authentication, cookies, multipart uploads, URL parsing, content/charset decoding, JSON, or environment-based configuration. The README frames this as "do one thing, and do it well" — the thing being connection management and byte transport.

Most developers never import httpcore directly. It exists as the transport layer beneath [httpx](https://www.python-httpx.org/): httpx handles the ergonomic client surface (redirects, auth, cookies, `response.json()`), and delegates the actual "open a connection, write bytes, read bytes" work to httpcore. The project's own README says as much — "you *probably* don't want to be using HTTP Core directly." Its target consumers are library and proxy authors who want a reusable pooling/transport core without a high-level client's opinions.

The defining design choice is the clean split between networking and protocol logic. httpcore drives the sans-I/O HTTP state machines (h11 for HTTP/1.1, h2 for HTTP/2) and owns everything about sockets, TLS, pooling, and concurrency around them. That separation is the whole point of the package's existence, and it is why the surface area is so small.

## Getting Started

```shell
# HTTP/1.1 only
pip install httpcore

# with async backends, HTTP/2, and SOCKS proxy support
pip install "httpcore[asyncio,trio,http2,socks]"
```

```python
import httpcore

# Convenience one-shot request (no pooling)
response = httpcore.request("GET", "https://www.example.com/")
print(response.status)        # 200
print(response.headers)       # [(b'Content-Type', b'text/html'), ...]
print(response.content)       # b'<!doctype html>...'

# In practice you want the connection pool
with httpcore.ConnectionPool() as http:
    r = http.request("GET", "https://www.example.com/")
    print(r.status)
```

```python
# Async: identical API under an AsyncConnectionPool
import httpcore

async def main():
    async with httpcore.AsyncConnectionPool() as http:
        r = await http.request("GET", "https://www.example.com/")
        print(r.status)
```

Note the byte-oriented interface: method, URL, and headers are `bytes`, not `str`. This is intentional — httpcore does no encoding decisions on your behalf.

## Architecture / How It Works

httpcore is built as a stack of small, swappable pieces:

- **`ConnectionPool` / `AsyncConnectionPool`** — the top-level entry point. Tracks a set of connections keyed by origin (scheme, host, port), enforces pool limits, hands out an idle connection or opens a new one, and reaps expired keep-alive connections. Thread-safe (sync) and task-safe (async).
- **`HTTPConnection`** — wraps a single transport connection and negotiates HTTP version. Over TLS it uses ALPN to decide between HTTP/1.1 and HTTP/2, then delegates to the matching connection type.
- **`HTTP11Connection` / `HTTP2Connection`** — drive the sans-I/O state machines. HTTP/1.1 uses `h11`; HTTP/2 uses `h2` (only installed with the `http2` extra). These libraries parse and serialize protocol bytes but never touch a socket themselves.
- **Network backends** — the actual I/O. httpcore ships a synchronous backend, an `anyio` backend (the path for `asyncio`), a `trio` backend, and a mock backend for tests. The async story is anyio-based: `pip install httpcore[asyncio]` pulls in `anyio`, and trio is supported directly[^2].

The sync and async code paths are largely a mirror of each other, historically kept in sync via code generation from the async source. Proxy support comes in two forms: forwarding proxies (for plain HTTP) and tunneling proxies via HTTP `CONNECT` (for HTTPS), plus SOCKS proxies through the optional `socksio` dependency.

Core non-optional dependencies are small: `certifi` for the CA bundle and `h11` for HTTP/1.1. Everything else — `anyio`, `trio`, `h2`, `socksio` — is an opt-in extra. This keeps the base install lean, which matters because httpcore sits deep in many dependency trees.

## Production Notes

**You are almost certainly using it transitively.** If you depend on httpx, you depend on httpcore. httpx pins httpcore to a narrow range, so bugs and behavior changes surface through httpx upgrades rather than direct control. When you see a connection-pool, keep-alive, or HTTP/2 issue in httpx, the root cause and the fix often live in this repo.

**Pool limits are the main tuning knob.** `ConnectionPool(max_connections=..., max_keepalive_connections=..., keepalive_expiry=...)` governs concurrency and connection reuse. The defaults are conservative; high-throughput services (and proxies, the intended direct-use case) usually need these raised. Exhausting `max_connections` blocks new requests until a connection frees up.

**HTTP/2 is opt-in and negotiated, not forced.** Without the `http2` extra installed, `h2` is absent and connections stay on HTTP/1.1 regardless of server support. HTTP/2 is selected via ALPN during the TLS handshake, so plaintext HTTP/2 (h2c) is not part of the normal path.

**The bytes-in/bytes-out contract is a footgun for direct users.** Passing `str` where `bytes` is expected, or expecting httpcore to decode gzip/brotli or follow a 302, will not work — those are httpx's job. This is by design, but it trips up anyone reaching for httpcore directly expecting a requests-like experience.

**Version pinning matters.** The project follows SemVer and recommends pinning to a major (`httpcore==1.*`). The 0.x line saw frequent interface churn; the 1.0 release (November 2023) stabilized the public API[^3]. Mixing an httpx version against an out-of-range httpcore is a common breakage — let httpx's pin drive the httpcore version rather than pinning httpcore independently.

## When to Use / When Not

**Use when:**
- You are building a library or client and want pooling, HTTP/1.1+HTTP/2, and sync+async transport without a high-level client's opinions.
- You are writing a proxy or gateway in Python that operates at the raw request/response level.
- You need to plug a custom transport into httpx (httpx transports are httpcore-shaped).

**Avoid when:**
- You just want to make HTTP calls in an app — use httpx or requests; httpcore gives you no redirects, auth, cookies, or JSON.
- You need websockets or an HTTP server — out of scope entirely.
- You want batteries-included async with its own high-level client — aiohttp fits that better.

## Alternatives

- encode/httpx — the high-level client built directly on httpcore; use this instead unless you truly need the raw transport layer.
- urllib3/urllib3 — the equivalent low-level layer beneath requests; use it when you're in a sync/requests-centric stack and don't need native async.
- aio-libs/aiohttp — async-first client and server with its own pool; use it when you want a batteries-included asyncio HTTP library rather than a minimal transport.
- python-hyper/h11 — the even-lower sans-I/O HTTP/1.1 state machine that httpcore itself uses; use it when you want to own all networking and only need protocol framing.
- psf/requests — the classic high-level sync client; use it for simple synchronous scripts where async and HTTP/2 don't matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2020-01 | Initial public release; low-level transport extracted for httpx[^1]. |
| 0.13 | 2021 | Major internal rework toward the current pool/backend architecture. |
| 0.16 | 2022 | Network backend abstraction consolidated (sync / anyio / trio / mock). |
| 1.0.0 | 2023-11 | First stable release; public API frozen under SemVer[^3]. |
| 1.x | 2024–2026 | Maintenance line; Python 3.8+ baseline, HTTP/2 and SOCKS as extras. |

## References

[^1]: encode/httpcore repository and README — "The HTTP Core package provides a minimal low-level HTTP client." https://github.com/encode/httpcore
[^2]: httpcore documentation — network backends and async support (asyncio via anyio, trio). https://www.encode.io/httpcore/
[^3]: httpcore CHANGELOG — 1.0 stabilization and SemVer policy. https://github.com/encode/httpcore/blob/master/CHANGELOG.md

## Tags

python, http-client, http2, connection-pooling, async, sans-io, transport-layer, networking, httpx, low-level
