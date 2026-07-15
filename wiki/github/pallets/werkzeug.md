# pallets/werkzeug

> The WSGI utility library that Flask is built on — request/response wrappers, URL routing, an interactive debugger, and a dev server, with no imposed framework.

[GitHub repo](https://github.com/pallets/werkzeug) ·
[Official website](https://werkzeug.palletsprojects.com) ·
[License: BSD-3-Clause](https://github.com/pallets/werkzeug/blob/main/LICENSE.txt)

## Overview

Werkzeug (German for "tool") is a collection of utilities for building WSGI-compliant web applications in Python[^1]. It started in 2007 as Armin Ronacher's grab-bag of helpers for the WSGI protocol (PEP 333/3333) and grew into the de facto standard library for the layer between a WSGI server and application code: parsing requests, building responses, matching URLs, and debugging. It is maintained by the Pallets organization alongside Flask, Jinja, Click, and ItsDangerous.

Most people use Werkzeug without knowing it — it is a hard dependency of Flask, which wraps it to provide app structure. Werkzeug itself imposes no structure: no template engine, no ORM, no configuration system. You assemble your own stack. That deliberate minimalism is its identity, but it also means Werkzeug is rarely used standalone in application code; the standalone consumers are mostly frameworks and libraries.

The defining tension is that Werkzeug is fundamentally **synchronous WSGI**. WSGI is a blocking, one-request-per-worker protocol. As the Python web world shifted toward ASGI and async I/O (Starlette, FastAPI, uvicorn), Werkzeug stayed anchored to the WSGI model. Version 2.0 added limited helpers for running `async` view functions, but the core request/response cycle is synchronous by design[^2]. For async-first workloads this is a hard architectural ceiling, and Pallets built a separate project, Quart, rather than retrofit Werkzeug.

## Getting Started

```bash
pip install Werkzeug
```

```python
# app.py — a WSGI application with no framework
from werkzeug.wrappers import Request, Response

@Request.application
def application(request: Request) -> Response:
    name = request.args.get("name", "World")
    return Response(f"Hello, {name}!")

if __name__ == "__main__":
    from werkzeug.serving import run_simple
    run_simple("127.0.0.1", 5000, application, use_reloader=True, use_debugger=True)
```

`run_simple` is a development server only. In production the same `application` callable is served by Gunicorn, uWSGI, or Waitress.

## Architecture / How It Works

Werkzeug is a set of loosely coupled modules over the WSGI contract — an `environ` dict in, a `(status, headers, body-iterable)` out via `start_response`. The notable pieces:

- **Wrappers** (`werkzeug.wrappers`) — `Request` reads the WSGI `environ` lazily (query args, form data, files, cookies, headers all parsed on first access); `Response` produces a WSGI-conformant iterable. These are the objects Flask subclasses.
- **Routing** (`werkzeug.routing`) — a `Map` of `Rule` objects compiled to regexes, bound to a request via `MapAdapter`. It does both matching (URL → endpoint) and building (endpoint + args → URL), with pluggable converters (`int`, `path`, custom). Flask's `@app.route` is a thin layer over this.
- **Data structures** (`werkzeug.datastructures`) — `MultiDict`, `Headers`, `EnvironHeaders`, `ImmutableMultiDict`, `FileStorage`. HTTP semantics (repeated keys, case-insensitive headers) live here, not in application code.
- **Local** (`werkzeug.local`) — `Local`, `LocalProxy`, `LocalStack`: context-local storage that Flask uses for `request`, `g`, and `session`. Originally thread-local, reimplemented on `contextvars` so it works under async and greenlets.
- **Debugger** (`werkzeug.debug`) — an in-browser interactive traceback: every frame in the stack becomes a live Python console. Protected by a PIN derived from machine attributes.
- **Serving** (`werkzeug.serving`) — `run_simple`, a threaded/forking WSGI server built on the stdlib `http.server`, plus a file-watching reloader (stat-based or watchdog-based).
- **Test** (`werkzeug.test`) — `Client` and `EnvironBuilder` synthesize `environ` dicts so requests can be exercised without a socket. This is the machinery behind Flask's `test_client()`.

The coupling to Flask runs one way: Werkzeug knows nothing about Flask, but Flask depends on Werkzeug's exact internal layout, so the two are released in close lockstep and Flask pins Werkzeug to a narrow version range.

## Production Notes

**`run_simple` is not a production server.** It is single-machine, has no process manager, and its own docs say so. Serve the WSGI callable with Gunicorn/uWSGI/Waitress behind Nginx.

**Never enable the debugger in production.** The interactive console is remote code execution by design. The PIN mitigates casual access but is not a real authentication boundary — CVE-2024-34069 showed the debugger could be reached when the app binds `0.0.0.0` and an attacker controls a sibling origin[^3]. Gate `use_debugger`/`debug=True` strictly to local development.

**Multipart and form parsing are DoS surfaces.** Werkzeug historically parsed unbounded numbers of form fields and file parts; CVE-2023-25577 (many small fields) and later multipart advisories led to hard limits. Set `Request.max_content_length`, `max_form_parts`, and `max_form_memory_size` — the defaults are conservative in recent versions but tune them for your endpoints[^4].

**Reverse-proxy headers are not trusted by default.** Behind a load balancer, wrap the app in `werkzeug.middleware.proxy_fix.ProxyFix` and configure exactly how many proxy hops to trust, or `request.remote_addr`/`url_scheme` will be wrong (and spoofable if you over-trust).

**Upgrades break downstream libraries.** Werkzeug removes long-deprecated APIs at major/minor boundaries. The 2.2→2.3 and 3.0 releases deleted helpers like the old `werkzeug.urls` quoting functions, which broke third-party Flask extensions that imported them; this is the classic "upgraded Werkzeug, Flask app won't import" failure[^5]. Pin Werkzeug together with Flask and read the changelog before bumping.

**The reloader runs your process twice.** With `use_reloader=True` the parent watches files and re-execs a child, so module-level side effects (opening ports, spawning threads) can fire twice. Guard them behind `WERKZEUG_RUN_MAIN`.

## When to Use / When Not

**Use when:**
- You are building your own WSGI framework or microframework and want routing, request/response objects, and a test client without adopting a full stack.
- You need correct, well-tested HTTP primitives (header parsing, content negotiation, cookies, ranges, etags) in an otherwise bespoke app.
- You are working inside Flask — you already depend on it.

**Avoid when:**
- Your workload is async/ASGI or high-concurrency I/O bound — WSGI's blocking model is the wrong fit; reach for Starlette/FastAPI or Quart.
- You want an actual framework with batteries — use Flask or Django rather than assembling Werkzeug by hand.
- You only need a production WSGI server — Werkzeug's serving module is for development, not deployment.

## Alternatives

- pallets/flask — the framework built directly on Werkzeug; use it when you want app structure, blueprints, and extensions instead of raw WSGI plumbing.
- pallets/quart — Flask's API reimplemented on ASGI; use it when you want Flask ergonomics but need real async.
- encode/starlette — lightweight ASGI toolkit (request/response/routing) that fills Werkzeug's niche for the async world.
- Pylons/webob — request and response objects for WSGI without the routing/debugger/server; use it when you only want the wrappers.
- benoitc/gunicorn — the WSGI server to run a Werkzeug/Flask app in production, replacing `run_simple`.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2007 | Initial release as a WSGI utilities collection[^1]. |
| 1.0 | 2020-02-06 | Dropped Python 2, restructured public API, deprecated legacy helpers. |
| 2.0 | 2021-05-11 | Type hints throughout, limited `async` view support, contextvars-based `Local`[^2]. |
| 2.3 | 2023-04 | Removed long-deprecated `werkzeug.urls`/`filesystem` helpers — common downstream break[^5]. |
| 3.0 | 2023-09-30 | Dropped older Python versions, further API cleanup, aligned with Flask 3. |
| 3.1 | 2024-11 | Continued 3.x line; current release series as of 2026. |

## References

[^1]: Werkzeug documentation — project overview and history. https://werkzeug.palletsprojects.com/en/stable/
[^2]: Werkzeug 2.0 changes, "Using Async and Await." https://werkzeug.palletsprojects.com/en/stable/request_data/ and https://palletsprojects.com/blog/werkzeug-2-0-0-released/
[^3]: GHSA / CVE-2024-34069 — Werkzeug debugger reachable via crafted request when binding to a public interface. https://github.com/pallets/werkzeug/security/advisories
[^4]: CVE-2023-25577 — Werkzeug high resource usage parsing multipart form data with many fields; `max_form_parts` limit. https://github.com/pallets/werkzeug/security/advisories
[^5]: Werkzeug CHANGES — removals in the 2.3 / 3.0 releases. https://werkzeug.palletsprojects.com/en/stable/changes/

## Tags

python, wsgi, http, web-framework-toolkit, routing, request-response, flask, pallets, dev-server, debugger, backend
