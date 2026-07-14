# aio-libs/aiohttp

> Asynchronous HTTP client and server framework built directly on Python's asyncio — the pre-ASGI async web stack that predates, and still competes with, the Starlette/uvicorn generation.

[GitHub repo](https://github.com/aio-libs/aiohttp) ·
[Official website](https://docs.aiohttp.org) ·
[License: Apache-2.0](https://github.com/aio-libs/aiohttp/blob/master/LICENSE.txt)

## Overview

aiohttp is one of the oldest still-active async HTTP libraries in the Python ecosystem, with first commits in 2013 — before `async`/`await` syntax existed (it originally used `asyncio` generator coroutines and `yield from`)[^1]. It bundles two loosely related things in one package: an async HTTP **client** (`aiohttp.ClientSession`) and an async HTTP **server** framework (`aiohttp.web`), both sitting directly on asyncio's transport/protocol layer.

The defining tension in 2026 is that aiohttp is **not ASGI**. It was designed and stabilized before the ASGI standard existed, and it implements its own request/response and lifecycle model rather than the `scope`/`receive`/`send` contract that Starlette, FastAPI, and uvicorn share. This means its server side does not slot into the modern ASGI ecosystem, and its middleware, routing, and deployment story are self-contained. New async web projects increasingly reach for FastAPI-on-uvicorn instead; aiohttp's **client**, however, remains widely used on its own merits, even inside otherwise-ASGI applications.

The audience is anyone doing high-concurrency HTTP I/O in Python who has already committed to asyncio: web scrapers and crawlers, service-to-service API gateways, WebSocket servers, and long-lived async services. It is a lower-level, less-batteries-included tool than the framework-first alternatives — you get HTTP, routing, and WebSockets, but validation, dependency injection, and OpenAPI generation are not in scope.

## Getting Started

```bash
pip install aiohttp
# recommended for non-blocking DNS resolution:
pip install aiohttp[speedups]   # pulls in aiodns, Brotli, cchardet-equivalent
```

```python
# Client — reuse one ClientSession for the whole program, not per request
import aiohttp
import asyncio

async def main():
    async with aiohttp.ClientSession() as session:
        async with session.get("https://python.org") as resp:
            print("Status:", resp.status)
            html = await resp.text()
            print("Body:", html[:15], "...")

asyncio.run(main())
```

```python
# Server — aiohttp.web, not ASGI
from aiohttp import web

async def handle(request):
    name = request.match_info.get("name", "Anonymous")
    return web.Response(text=f"Hello, {name}")

app = web.Application()
app.add_routes([web.get("/", handle), web.get("/{name}", handle)])
web.run_app(app)  # binds its own server; no external ASGI server needed
```

## Architecture / How It Works

**Client.** `ClientSession` owns a `TCPConnector`, which holds the connection pool, keep-alive connections, DNS cache, and TLS context. A session is meant to live for the lifetime of the application — creating one per request throws away pooling and keep-alive entirely, which is the single most common misuse. Requests return a `ClientResponse` whose body must be consumed or released; the `async with` form handles this. Redirects, cookies, and connection reuse are handled at the session level.

**Server.** `web.Application` is a mapping-like container with a `UrlDispatcher` for routing, a middleware chain, and lifecycle signals (`on_startup`, `on_cleanup`, `on_shutdown`). A request is parsed into a `web.Request`, passed through middlewares (plain `async` functions wrapping a handler), and the handler returns a `web.Response` or `web.StreamResponse`. Runners (`AppRunner` + `TCPSite`) let you embed the server in an existing event loop instead of using `run_app`.

**HTTP parsing.** aiohttp ships C extensions for the HTTP parser and other hot paths. Since 3.8 the parser is based on **llhttp** (the same parser family Node.js uses), replacing the older http-parser[^2]. A pure-Python fallback exists and is selected automatically when the C extensions aren't built, or forced via the `AIOHTTP_NO_EXTENSIONS=1` environment variable — useful on platforms without a compiler, at a real throughput cost.

**WebSockets.** Both client (`session.ws_connect`) and server (`web.WebSocketResponse`) are first-class and share the framing implementation. This is one area where aiohttp is genuinely more complete out-of-the-box than several ASGI frameworks.

**DNS.** By default the standard-library resolver runs in a thread executor. Installing `aiodns` (which wraps c-ares) enables a fully async resolver — recommended for any workload opening many distinct hosts, since the threaded default can bottleneck.

## Production Notes

- **One session, reused.** Instantiate `ClientSession` once (e.g. in `on_startup`, stored on the app) and close it in `on_cleanup`. Per-request sessions are the top footgun and produce `Unclosed client session` / `Unclosed connector` warnings at shutdown.
- **Connector limits.** `TCPConnector` defaults to `limit=100` total connections and `limit_per_host=0` (unlimited per host). Under a burst against one host you can exhaust the total pool or overwhelm the target; tune both explicitly for crawlers.
- **Timeouts.** The default `ClientTimeout.total` is 300 seconds — long enough that a hung upstream stalls a task for five minutes silently. Set `timeout=aiohttp.ClientTimeout(total=..., connect=...)` per session or per request.
- **Blocking the loop kills it.** Like all asyncio code, any synchronous/CPU-bound call inside a handler blocks every other in-flight request. Offload to executors or separate processes.
- **Deployment.** There is no ASGI server to point at it. Production options are `web.run_app`, the `gunicorn` worker (`aiohttp.GunicornWebWorker` / `GunicornUVLoopWebWorker`), or a hand-rolled `AppRunner`. Put it behind nginx or another reverse proxy for TLS termination and static files; aiohttp's static file serving is fine for development but not a CDN substitute.
- **uvloop.** Swapping in `uvloop` as the event loop policy is a common and low-effort throughput win for both client and server workloads.
- **Not ASGI, both ways.** You cannot mount an aiohttp app under uvicorn/Hypercorn, and you cannot mount Starlette/FastAPI sub-apps inside aiohttp. Mixing ecosystems means running two servers.
- **The perpetual 4.0.** A 4.0 release has been in alpha/pre-release for years; most production usage is on the stable 3.x line. Treat 4.0 as not-yet-shipped when planning.

## When to Use / When Not

**Use when:**
- You need a high-concurrency async HTTP **client** with real connection pooling (scrapers, crawlers, fan-out API aggregation).
- You want client + server + WebSockets from one dependency without adopting the full ASGI stack.
- You already have an asyncio codebase and want a mature, battle-tested HTTP layer.

**Avoid when:**
- You're starting a new typed web API — FastAPI/Starlette on uvicorn is the mainstream path with validation, OpenAPI, and dependency injection built in.
- You need ASGI interop (mounting under uvicorn, sharing middleware with Starlette apps).
- You only make occasional HTTP calls and don't need async — `requests` or `httpx` (sync mode) is simpler.
- Your team is new to asyncio; the client's session lifecycle and event-loop discipline have a real learning curve.

## Alternatives

- encode/httpx — use instead when you want a requests-like client API, sync **and** async in one library, and better fit alongside ASGI apps; slightly slower under extreme concurrency but far friendlier.
- encode/starlette — use instead when building a new async web server that must be ASGI-native and composable with the uvicorn ecosystem.
- tiangolo/fastapi — use instead when you want typed request/response models, validation, and OpenAPI generation on top of Starlette.
- psf/requests — use instead when you don't need async at all and want the most familiar synchronous client.
- tornadoweb/tornado — use instead only when maintaining an existing Tornado codebase; it overlaps aiohttp's niche but has less momentum.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-10 | First commits, pre-`async`/`await`, built on asyncio coroutines[^1]. |
| 1.0 | 2016-10 | First stable line; client + `aiohttp.web` server. |
| 2.0 | 2017-03 | API cleanup; `ClientSession`-centric client model solidified. |
| 3.0 | 2018-02 | Major release; dropped older Python, `async`/`await`-native throughout. |
| 3.8 | 2021-11 | Switched HTTP parser to llhttp[^2]. |
| 3.9 | 2023-11 | Python 3.12 support; continued 3.x maintenance line. |
| 3.10 | 2024 | Performance and security fixes; middleware/runtime refinements[^3]. |
| 3.11 | 2024-11 | Latest stable 3.x series; 4.0 still in pre-release[^3]. |

## References

[^1]: aiohttp repository history and About page — created 2013-10-01. https://github.com/aio-libs/aiohttp
[^2]: aiohttp CHANGES — 3.8 series moved the C HTTP parser to llhttp. https://docs.aiohttp.org/en/stable/changes.html
[^3]: aiohttp changelog / PyPI release history. https://pypi.org/project/aiohttp/#history
[^4]: Official documentation (client usage, `ClientSession`, deployment). https://docs.aiohttp.org/

## Tags

python, asyncio, async, http-client, http-server, websockets, networking, web-framework, non-asgi, aio-libs, apache-2.0
