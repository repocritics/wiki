# pallets/flask

> The Python micro-framework — a thin, unopinionated WSGI layer over Werkzeug and Jinja that you assemble into an application yourself.

[GitHub repo](https://github.com/pallets/flask) ·
[Official website](https://flask.palletsprojects.com) ·
[License: BSD-3-Clause](https://github.com/pallets/flask/blob/main/LICENSE.txt)

## Overview

Flask is a lightweight WSGI web framework by Armin Ronacher, first released in 2010 and now maintained by the Pallets organization[^1]. It began as a wrapper tying together two of Ronacher's existing libraries — Werkzeug (a WSGI request/response and routing toolkit) and Jinja (a templating engine) — and has stayed deliberately small ever since. The "micro" in micro-framework refers to scope, not capability: Flask ships routing, request handling, templating integration, a signed-cookie session, a development server, and a CLI, and nothing else. There is no ORM, no form validation, no authentication, no admin, and no prescribed project layout.

That minimalism is the defining tradeoff. Flask hands the developer a clean core and expects them to choose and wire up every other component (database, migrations, auth, serialization) from the community extension ecosystem or the wider Python world. This makes Flask excellent for small services and for teams who want to understand every moving part, and a liability for large applications that end up re-implementing — inconsistently, project by project — the conventions that a batteries-included framework would have supplied.

Flask is synchronous and WSGI-based at its core. It gained `async def` view support in 2.0 (2021), but this runs async views inside a worker thread rather than on an event loop, so it is a compatibility feature, not a path to high-concurrency async I/O[^2]. Teams that need genuine ASGI/async should reach for Quart (the Pallets async re-implementation of the Flask API) or another framework.

## Getting Started

```bash
pip install Flask
```

```python
# app.py
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, World!"

@app.route("/echo", methods=["POST"])
def echo():
    data = request.get_json()
    return jsonify(received=data), 200
```

```bash
flask --app app run --debug   # dev server on http://127.0.0.1:5000
```

The `flask run` development server is single-process and single-threaded by default; it exists for local iteration only and must not serve production traffic.

## Architecture / How It Works

Flask is a coordination layer over three Pallets libraries, and understanding it is mostly understanding that delegation:

- **Werkzeug** provides the WSGI plumbing — the `Request`/`Response` objects, URL routing (`Map`/`Rule`), the interactive debugger, and the dev server. Flask's routing decorators are a thin façade over Werkzeug's `Map`.
- **Jinja** provides `render_template` and the templating environment, auto-configured to look in the app's `templates/` folder with autoescaping on for `.html`.
- **Click** provides the `flask` CLI and the `@app.cli.command()` decorator.
- **itsdangerous** cryptographically signs the session cookie.

**Contexts and proxies.** Flask exposes `request`, `session`, `g`, and `current_app` as importable globals, but they are context-local proxies (`LocalProxy`), not real objects. Two stacks — the *application context* and the *request context* — are pushed at the start of each request and popped at the end. Since 2.0 these are backed by `contextvars` rather than thread-locals, which is what allows async views to work at all. The practical consequence: outside a request (background threads, CLI scripts, startup code) those proxies raise `RuntimeError: working outside of application context` unless you manually enter `with app.app_context():`.

**Blueprints** are Flask's modularity unit — a group of routes, templates, and static files registered onto an app under a URL prefix, allowing an application to be split across files without circular-import gymnastics. There is no deeper module system; large apps are just many blueprints plus an application factory (`create_app()`).

**Sessions** are client-side by default: the entire session dict is serialized, signed with `SECRET_KEY`, and stored in the user's cookie. It is signed, not encrypted — the contents are readable by the client. This is the single most misunderstood part of Flask.

## Production Notes

**Never run the dev server in production.** Deploy behind a real WSGI server — Gunicorn or uWSGI — typically with Nginx in front. Concurrency comes from the number of worker processes/threads you configure, because a synchronous WSGI worker handles exactly one request at a time.

**The sync/concurrency ceiling.** A CPU-bound or slow-I/O request occupies a whole worker for its duration. Scaling an I/O-heavy Flask app means many workers, threaded workers, or a `gevent`/`eventlet` worker class that monkey-patches the stack for greenlet concurrency. None of these give you the clean async story of an ASGI framework; if your workload is fundamentally async (websockets, thousands of idle long-poll connections), Flask is the wrong tool.

**Sessions store no secrets and are size-limited.** Because the default session is a signed cookie, don't put anything confidential in it, and stay under the ~4KB cookie limit. Rotating `SECRET_KEY` invalidates every existing session. Server-side sessions require the Flask-Session extension or a hand-rolled store.

**`app.run()` and background work.** Threads or schedulers you spawn yourself run outside any request/app context; they must push `app.app_context()` before touching `current_app`, the database, or config. This is a recurring source of `RuntimeError` in production code.

**Extension quality varies.** Core plumbing (Flask-SQLAlchemy, Flask-Login, Flask-Migrate, Flask-WTF) is mature and well-maintained; the long tail of Flask-* packages ranges from solid to abandoned. Because Flask prescribes nothing, two Flask codebases can look nothing alike, which raises the onboarding cost of an unfamiliar app.

**Debug mode is dangerous if exposed.** `debug=True` enables the Werkzeug interactive debugger, which can execute arbitrary code in the browser; it is PIN-protected but must never be reachable in production.

**Upgrade pains are modest.** Flask has been unusually stable. The notable breaks: 2.0 dropped Python 2 and 3.5 and moved contexts to `contextvars`; 2.3 removed several long-deprecated APIs (including the old `flask.json` provider hooks and `app.run` defaults); 3.0 dropped Python 3.7 and required a modern Werkzeug. Most upgrades are uneventful, but Werkzeug's version is effectively pinned to Flask's — mismatches are a common cause of import errors.

## When to Use / When Not

**Use when:**
- You want a small, legible service or API and prefer choosing your own components.
- You need a WSGI app that any Python host or PaaS will run without special support.
- You're teaching or learning web fundamentals — Flask hides very little.
- You're adding a modest HTTP surface to an existing synchronous Python codebase.

**Avoid when:**
- You want batteries included (ORM, admin, auth, migrations) out of the box — Django is the answer.
- Your workload is async-native: websockets, high-concurrency streaming, or thousands of idle connections.
- You want automatic request/response validation and OpenAPI docs from type hints — FastAPI covers this directly.
- You're building a large team application and want enforced conventions rather than per-project improvisation.

## Alternatives

- django/django — batteries-included; use when you want ORM, admin, auth, and migrations rather than assembling them.
- tiangolo/fastapi — async-native with Pydantic validation and automatic OpenAPI; use when building typed JSON APIs.
- pallets/quart — near-identical Flask API on ASGI; use when you need real async, websockets, or HTTP/2 with minimal code change.
- encode/starlette — lower-level ASGI toolkit; use when you want FastAPI's foundation without the batteries.
- bottlepy/bottle — single-file WSGI micro-framework; use when you want zero dependencies and an even smaller surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2010-04 | Initial release by Armin Ronacher over Werkzeug + Jinja[^1]. |
| 0.12 | 2016-12 | Last of the long 0.x line; `flask` CLI introduced (0.11). |
| 1.0 | 2018-04 | First stable major; dropped some legacy behavior, CLI matured. |
| 1.1 | 2019-07 | JSON and config refinements. |
| 2.0 | 2021-05 | `async`/`await` views, `contextvars` contexts, Python 2 dropped[^2]. |
| 2.2 | 2022-08 | Faster routing via Werkzeug matcher, typing improvements. |
| 2.3 | 2023-04 | Removed long-deprecated APIs; Python 3.8+ required. |
| 3.0 | 2023-09 | Dropped Python 3.7; modern Werkzeug baseline[^3]. |
| 3.1 | 2024-11 | Session/config refinements, updated dependencies. |

## References

[^1]: Pallets, "Flask Documentation — Foreword / Design Decisions". https://flask.palletsprojects.com/en/stable/design/
[^2]: Pallets, "Using async and await" (Flask 2.0). https://flask.palletsprojects.com/en/stable/async-await/
[^3]: Pallets, "Flask Changes / Changelog". https://flask.palletsprojects.com/en/stable/changes/

## Tags

python, web-framework, wsgi, micro-framework, backend, werkzeug, jinja, pallets, rest-api, synchronous
