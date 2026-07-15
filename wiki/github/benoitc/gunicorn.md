# benoitc/gunicorn

> A pre-fork WSGI HTTP server for UNIX — the boring, dependable process manager that has run Python web apps behind nginx since 2010.

[GitHub repo](https://github.com/benoitc/gunicorn) ·
[Official website](https://gunicorn.org) ·
[License: MIT](https://github.com/benoitc/gunicorn/blob/master/LICENSE)

## Overview

Gunicorn ("Green Unicorn") is a WSGI HTTP server for UNIX, first released in 2010 as a port of Ruby's Unicorn pre-fork model to Python[^1]. Its design goal was narrow and it has stayed narrow: accept HTTP, hand requests to a WSGI (and now ASGI) application, and manage a pool of worker processes robustly. It does not do TLS termination well, does not buffer slow clients well on its default worker, and is not intended to face the public internet alone — it expects a reverse proxy in front.

The defining tension is between simplicity and scope. Gunicorn deliberately refuses the feature sprawl of uWSGI: no built-in caching, cron, or routing, minimal config surface, and a single obvious job. That makes it easy to reason about and hard to misconfigure catastrophically, but it also means the operator owns the parts Gunicorn declines — a buffering proxy, process supervision, and a worker-class choice that matches the workload. Picking the wrong worker class is the most common source of "Gunicorn is slow" complaints, and it is almost always an application/topology problem rather than a server one.

At ~10.6k stars and 16 years of history, Gunicorn is mature infrastructure maintained largely by volunteers[^2]. The commit cadence is slow but real; the README now advertises native ASGI, HTTP/2 (beta), and "Dirty Arbiters" for heavy workloads as newer additions[^3], expanding a project that was WSGI-only for most of its life.

## Getting Started

```bash
pip install gunicorn
```

```bash
# Serve a WSGI app (Django/Flask/Pyramid). module:callable
gunicorn myapp:app --workers 4 --bind 0.0.0.0:8000
```

```python
# gunicorn.conf.py — configuration as code, auto-loaded from CWD
workers = 4                 # rule of thumb: (2 * cores) + 1
worker_class = "gthread"    # threaded worker for I/O-bound apps
threads = 4
timeout = 30                # worker killed if a request exceeds this
max_requests = 1000         # recycle workers to bound memory growth
max_requests_jitter = 100
preload_app = True          # load app before fork (copy-on-write memory)
bind = "0.0.0.0:8000"
```

```bash
gunicorn myapp:app -c gunicorn.conf.py
```

## Architecture / How It Works

Gunicorn runs one **arbiter** (master) process that forks and supervises N **worker** processes. All workers inherit the same listening socket and call `accept()` on it; the OS arbitrates which worker wins each connection, so there is no user-space load balancer. The arbiter never touches request bytes — it only manages worker lifecycle.

Liveness is tracked through a heartbeat: each worker periodically touches a temp file, and the arbiter kills and replaces any worker whose heartbeat is older than `timeout` (default 30s). This is why long-running requests get killed mid-flight on the default worker — the same mechanism that recovers from hung workers also punishes slow ones.

Worker classes are the real architectural lever:

- **sync** — one request at a time per worker, fully blocking. Simple, memory-cheap, and safe, but a single slow client ties up a whole worker. Must sit behind a buffering proxy.
- **gthread** — a worker with a thread pool; good for I/O-bound apps that hold the GIL only briefly.
- **gevent / eventlet** — greenlet-based async via monkey-patching; high connection concurrency for I/O-bound workloads, at the cost of debuggability and library compatibility.
- **asgi** (newer) — native ASGI support for FastAPI/Starlette/Quart[^3]. Historically this role was filled by running Uvicorn's worker class under Gunicorn (`uvicorn.workers.UvicornWorker`), using Gunicorn purely as a process manager.

Signals are the control plane: `HUP` reloads config and gracefully restarts workers, `TTIN`/`TTOU` add/remove workers live, `USR2` performs a zero-downtime binary upgrade by forking a new master, and `TERM`/`QUIT` shut down. Because it relies on `fork()`, Gunicorn is UNIX-only — there is no native Windows support.

## Production Notes

**Put a buffering reverse proxy in front.** The sync worker reads the request directly from the client socket, so a slow or malicious client (slowloris) can occupy a worker indefinitely. nginx (or any buffering proxy) absorbs slow clients and only hands Gunicorn complete requests. This is not optional advice for public deployments — it is the intended topology.

**The `timeout` footgun.** `timeout` is a worker-liveness watchdog, not an HTTP request deadline. Setting it low to "fail fast" will kill legitimate long requests (large uploads, report generation, streaming). For genuinely long work, raise `timeout`, move to an async worker, or push the work to a queue — do not fight the watchdog.

**Bound worker memory.** Python web apps leak slowly. `max_requests` + `max_requests_jitter` recycle workers after a request count (jitter avoids a synchronized restart stampede). `preload_app = True` loads the app once in the arbiter so forked workers share pages via copy-on-write, cutting memory — but it disables code auto-reload and runs import-time side effects before the fork, so anything holding a file descriptor or DB connection at import time must be re-initialized in a `post_fork` hook.

**Worker count is not "more is better."** Each sync/gthread worker is a full process; the `(2 * cores) + 1` heuristic exists because more workers than that mostly adds memory and context-switching, not throughput. CPU-bound apps scale with processes; I/O-bound apps scale better with threads or async workers than with raw worker count.

**HTTP parsing history.** Gunicorn's lenient request parsing has produced request-smuggling vulnerabilities; 22.0.0 (2024) shipped fixes for `Transfer-Encoding` handling (CVE-2024-1135)[^4]. Keep it current and terminate untrusted HTTP at a strict proxy.

**Supervision.** Gunicorn manages workers, not itself. Run the arbiter under systemd, or a container orchestrator — Gunicorn will not restart its own master if it dies.

## When to Use / When Not

**Use when:**
- You run a synchronous WSGI app (Django, Flask, Pyramid) on Linux behind nginx.
- You want a small, predictable config surface and battle-tested process management.
- You need zero-downtime reloads/upgrades via signals without extra tooling.

**Avoid when:**
- You deploy on Windows — Gunicorn requires `fork()`; use Waitress or Uvicorn instead.
- Your app is purely async and you want the leanest stack — a bare Uvicorn/Hypercorn may suffice.
- You need built-in routing, caching, or a Swiss-army feature set — that is uWSGI's territory, by choice not Gunicorn's.

## Alternatives

- encode/uvicorn — the reference ASGI server; use for async apps (FastAPI/Starlette), often as Gunicorn's worker or standalone.
- unbit/uwsgi — heavier, feature-maximal C server; use when you want caching/cron/routing built in and accept the config complexity.
- Pylons/waitress — pure-Python WSGI server; use when you need Windows support or a zero-C-dependency deploy.
- pgjones/hypercorn — ASGI server with first-class HTTP/2 and HTTP/3; use when you need those protocols natively.
- emmett-framework/granian — Rust-based Python server (WSGI/ASGI/RSGI); use when you want a modern single-binary alternative.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2010-01 | Initial release; pre-fork model ported from Ruby Unicorn[^1]. |
| 19.0 | 2014-06 | Long-lived 19.x series; broad framework adoption. |
| 20.0 | 2019-11 | Dropped Python 2; Python 3-only from here[^5]. |
| 21.0 | 2023-07 | Python 3.11 support, packaging/tooling refresh. |
| 22.0 | 2024-04 | Request-smuggling fixes, `Transfer-Encoding` hardening (CVE-2024-1135)[^4]. |
| 23.0 | 2024-08 | Maintenance release; config and worker fixes. |
| 25.x | 2026 | Native ASGI worker, HTTP/2 (beta), Dirty Arbiters per README[^3]. |

## References

[^1]: Gunicorn design docs — pre-fork model, ported from Ruby's Unicorn. https://docs.gunicorn.org/en/stable/design.html
[^2]: GitHub repository metadata (stars, forks, last push) — fetched 2026-07-15. https://github.com/benoitc/gunicorn
[^3]: Gunicorn README — ASGI, HTTP/2 (beta), and Dirty Arbiters feature notes. https://github.com/benoitc/gunicorn/blob/master/README.rst
[^4]: CVE-2024-1135 — Gunicorn HTTP request smuggling via Transfer-Encoding; fixed in 22.0.0. https://nvd.nist.gov/vuln/detail/CVE-2024-1135
[^5]: Gunicorn 20.0 release notes — Python 2 removed. https://docs.gunicorn.org/en/stable/news.html

## Tags

python, wsgi, asgi, http-server, web-server, pre-fork, process-manager, unix, deployment, gunicorn
