# tornadoweb/tornado

> A single-threaded, non-blocking Python web framework and networking library that predates asyncio — and now runs on top of it.

[GitHub repo](https://github.com/tornadoweb/tornado) ·
[Official website](http://www.tornadoweb.org/) ·
[License: Apache-2.0](https://github.com/tornadoweb/tornado/blob/master/LICENSE)

## Overview

Tornado is a Python web framework combined with an asynchronous networking library. It was written at FriendFeed and open-sourced by Facebook in 2009 after the acquisition[^1]. Its design goal was to hold tens of thousands of simultaneous open connections on a single thread using non-blocking I/O — the "C10k" problem — which made it a natural fit for long polling, WebSockets, and other long-lived-connection workloads at a time when the standard Python answer was a thread-or-process per request.

Its defining tension is historical. Tornado shipped its own event loop (`IOLoop`), its own coroutine system (`@gen.coroutine` + `yield`), and its own HTTP server years before Python had `asyncio` (added in 3.4, 2014) or `async`/`await` syntax (3.5, 2015). When those landed in the standard library, Tornado had to decide whether to stay a parallel universe or converge. It converged: since Tornado 5 (2018) the `IOLoop` is a thin wrapper over the asyncio event loop, and since Tornado 6 (2019) native `async`/`await` is the only supported coroutine style[^2][^3]. The result is a mature, stable, batteries-included stack that still carries visible seams from its pre-asyncio origins.

As of 2026 the repository remains actively maintained (last push 2026-07) with ~22k stars, but its mindshare for new projects has been eroded by the ASGI ecosystem (Starlette, FastAPI, uvicorn) that grew up asyncio-native. Tornado is now most valuable to existing Tornado codebases and to teams that specifically want a self-contained server-plus-framework in one dependency.

## Getting Started

```bash
pip install tornado
```

```python
import asyncio
import tornado

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")

class NameHandler(tornado.web.RequestHandler):
    async def get(self, name):
        # await any coroutine here; the loop stays free for other requests
        self.write(f"Hello, {name}")

def make_app():
    return tornado.web.Application([
        (r"/", MainHandler),
        (r"/hello/([^/]+)", NameHandler),
    ])

async def main():
    app = make_app()
    app.listen(8888)
    await asyncio.Event().wait()   # run forever

if __name__ == "__main__":
    asyncio.run(main())
```

The routing table maps URL regexes to `RequestHandler` subclasses; handler methods are named after HTTP verbs (`get`, `post`, ...) and may be `async`.

## Architecture / How It Works

Tornado is four layers that ship together:

1. **`IOLoop`** — the event loop. Since 5.0 it delegates to `asyncio`; `IOLoop.current()` returns an object bound to the running asyncio loop. This is why Tornado apps and pure-asyncio libraries can share one loop.
2. **`IOStream` / `TCPServer` / `HTTPServer`** — non-blocking socket handling and a from-scratch HTTP/1.x implementation. Tornado's server is its own; it is not a WSGI or ASGI server.
3. **`tornado.web`** — the framework: `Application` (routing, settings), `RequestHandler` (request lifecycle, `write`/`render`/`redirect`, XSRF, secure cookies), and the `tornado.template` engine.
4. **Client + protocol libraries** — `AsyncHTTPClient` (`SimpleAsyncHTTPClient` in pure Python, or `CurlAsyncHTTPClient` backed by libcurl for richer proxy/HTTP features), `tornado.websocket`, `tornado.gen`, and `tornado.queues`.

The concurrency model is a single-threaded cooperative loop: one process handles all connections by never blocking. Any `await`-point yields control back to the loop. Legacy `tornado.gen.coroutine`/`yield` code still runs (it is bridged to awaitables) but new code should use native coroutines.

Because Tornado predates ASGI, it does **not** implement the ASGI interface. You cannot drop a Tornado app under uvicorn, and Starlette/FastAPI middleware does not compose with it. Tornado can, however, host asyncio libraries directly, and `tornado.wsgi.WSGIContainer` can wrap a synchronous WSGI app (at the cost of the async benefits).

## Production Notes

**The single-threaded loop is the whole story.** Any blocking call — a CPU-bound computation, a synchronous DB driver (`psycopg2`, a blocking `requests` call), `time.sleep`, a large `json.loads` — stalls *every* connection on that process, not just the current request. The two escapes are: use async-native libraries, or push blocking work to `IOLoop.run_in_executor` / a `ThreadPoolExecutor`. This is the most common Tornado outage cause.

**Scaling across cores is manual.** One Tornado process uses one core. Production deployments run N processes (commonly one per core) behind a reverse proxy such as nginx, either via `tornado.process.fork_processes` / `HTTPServer.start(n)` or an external supervisor. There is no built-in multi-process load balancing beyond the fork helper, and forking interacts awkwardly with sockets already bound to an asyncio loop, so the `bind` + `fork` + `add_sockets` ordering matters.

**Choose the HTTP client deliberately.** `SimpleAsyncHTTPClient` has no external dependency but a limited feature set; `CurlAsyncHTTPClient` needs `pycurl`/libcurl but supports proxies, some auth schemes, and behaviors the pure-Python client omits. The client is a process-wide singleton by default, so per-request configuration is a known gotcha.

**No ORM, no admin, no auth backend.** Tornado is closer to Flask than Django in scope: routing, templating, request handling, WebSockets, secure cookies, and XSRF are included; persistence, migrations, forms, and authentication are your problem. `RequestHandler.get_current_user` is a hook, not an implementation.

**Upgrade history worth knowing.** The 4.x→5.x jump moved onto asyncio (behavioral changes around loop ownership). The 5.x→6.x jump dropped Python 2 and removed callback-style APIs in favor of coroutines, breaking older code that passed callbacks. Recent 6.x releases have steadily raised the minimum Python version, so pinning `tornado<6` or a specific 6.x is common in legacy stacks.

## When to Use / When Not

**Use when:**
- You maintain or extend an existing Tornado application.
- You need long-lived connections — WebSockets, long polling, SSE, streaming — in a single self-contained package.
- You want one dependency that is both the server and the framework, with no separate ASGI server to operate.
- You need to embed a web endpoint inside an existing asyncio program and want tight control over the loop.

**Avoid when:**
- You are starting a new API/service and want the current mainstream path — an ASGI stack (FastAPI/Starlette on uvicorn) has more ecosystem, middleware, and type-driven tooling.
- Your workload is CPU-bound; the single-threaded loop fights you unless you offload aggressively.
- You want a full-stack framework with ORM, migrations, admin, and auth out of the box (Django).
- You need ASGI middleware/interoperability — Tornado does not speak ASGI.

## Alternatives

- aio-libs/aiohttp — closest peer: async HTTP client + server on asyncio; use instead when you want an asyncio-native library with a similar "server and client in one" shape but no legacy coroutine baggage.
- encode/starlette — lightweight ASGI toolkit; use instead when you want modern middleware and ASGI-server interoperability for a new project.
- fastapi/fastapi — API framework on Starlette with type-driven validation and OpenAPI; use instead when building JSON APIs and you want schema/docs for free.
- pallets/flask — synchronous micro-framework; use instead when you have no real concurrency need and want the largest extension ecosystem.
- django/django — full-stack batteries-included framework; use instead when you want ORM, admin, auth, and migrations rather than assembling them.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2009-09 | Open-sourced by Facebook, from FriendFeed's stack[^1]. |
| 3.0 | 2013-03 | Futures-based coroutines (`tornado.gen`, `tornado.concurrent`). |
| 4.0 | 2014-07 | Coroutine-centric rewrite of the async model; WSGI support reworked. |
| 5.0 | 2018-03 | Integrated with `asyncio`; `IOLoop` becomes an asyncio wrapper[^2]. |
| 6.0 | 2019-03 | Dropped Python 2; native `async`/`await` only, callback APIs removed[^3]. |
| 6.2 | 2022-07 | Raised minimum Python; deprecations toward modern asyncio. |
| 6.4 | 2024-03 | Python 3.8+ line; maintenance and security fixes. |

## References

[^1]: Bret Taylor, "Tornado: Facebook's Real-Time Web Framework for Python" — 2009-09-10. https://www.facebook.com/notes/facebook-engineering/tornado-facebooks-real-time-web-framework-for-python/450061798919/
[^2]: Tornado documentation, "What's new in Tornado 5.0" — asyncio integration. https://www.tornadoweb.org/en/stable/releases/v5.0.0.html
[^3]: Tornado documentation, "What's new in Tornado 6.0" — Python 3 only, async/await. https://www.tornadoweb.org/en/stable/releases/v6.0.0.html

## Tags

python, web-framework, asynchronous, networking, websockets, event-loop, asyncio, non-blocking-io, long-polling, http-server
