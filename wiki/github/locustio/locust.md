# locustio/locust

> Load testing tool where the test plan is plain Python and every simulated user is a gevent greenlet.

[GitHub repo](https://github.com/locustio/locust) ·
[Official website](https://locust.cloud) ·
[License: MIT](https://github.com/locustio/locust/blob/master/LICENSE)

## Overview

Locust is an open-source performance/load testing tool for HTTP and other protocols, first released in 2011 by Jonatan Heyman[^1]. Its defining choice is that a test plan is ordinary Python code: you subclass `HttpUser`, decorate methods with `@task`, and the framework runs many copies of that user concurrently. There is no GUI-authored scenario, no XML, and no domain-specific language — the tradeoff Locust makes against tools like JMeter is expressiveness and version-controllability over point-and-click accessibility.

The concurrency model is the other defining decision. Each simulated user runs inside its own greenlet (a cooperative coroutine) on top of gevent, so a single Python process can hold tens of thousands of users open cheaply, provided the workload is I/O-bound. This is what lets Locust describe blocking-looking code (`self.client.get(...)`) that is actually non-blocking under the hood. The cost is that raw requests-per-second on one core is lower than compiled tools, and CPU-bound test logic breaks the model.

As of 2026 the project is heavily used and actively maintained — roughly 28k stars, over 3,200 forks[^2], with commits landing daily. It is maintained primarily by Lars Holmberg, and the `locust.cloud` domain (the repo's current homepage) is a commercial managed-run offering from the same maintainers; the core tool remains MIT-licensed and self-hostable with no cloud dependency.

## Getting Started

```bash
pip install locust
locust --version
```

Write a `locustfile.py`:

```python
from locust import HttpUser, task, between

class QuickstartUser(HttpUser):
    wait_time = between(1, 2)          # each user idles 1–2s between tasks

    def on_start(self):
        self.client.post("/login", json={"username": "foo", "password": "bar"})

    @task(3)                            # weight 3 — picked 3x as often
    def view_items(self):
        for item_id in range(10):
            self.client.get(f"/item?id={item_id}", name="/item")   # group under one label

    @task
    def about(self):
        self.client.get("/about")
```

Run with the web UI (defaults to http://localhost:8089), or headless for CI:

```bash
locust -f locustfile.py --host https://example.com          # opens web UI
locust -f locustfile.py --host https://example.com \
       --headless -u 500 -r 50 --run-time 5m --csv results   # 500 users, spawn 50/s
```

## Architecture / How It Works

**Greenlets over gevent.** At import time Locust monkey-patches the standard library (socket, ssl, threading) via gevent so that blocking calls yield to the scheduler. Every user instance and its wait loop is a greenlet. This is why a `locustfile` must not do heavy CPU work or call genuinely blocking C extensions in a task — one uncooperative greenlet stalls every other user in that process.

**The `HttpUser.client`** is a subclass of a `requests.Session` that records timing, groups results by the `name` argument, and reports failures for non-2xx responses. Because it is `requests`-based it is convenient but comparatively CPU-heavy per request. `FastHttpUser` is the alternative client, built on geventhttpclient, and delivers substantially higher throughput per core at the cost of a slightly different (less feature-complete) request API — this swap is the single most common performance lever in real Locust deployments.

**Distributed mode is master + workers.** The master process runs the web UI and aggregates statistics but generates no load itself. Worker processes connect to the master over ZeroMQ and actually simulate the users; the master tells each worker how many users to spawn and collects their stats. Because one process is bound by a single CPU core (gevent is cooperative, and Python's GIL applies), the standard scaling recipe is to run one worker per core — even on a single machine — rather than one big process.

**Extensibility is by design and by omission.** The codebase is deliberately small and does not try to cover every protocol. Custom protocols are supported by writing your own client and firing the `request` event so timings show up in the stats; `LoadTestShape` subclasses let you script arbitrary user-count-over-time profiles; and an event system (`events.request`, `events.init`, `events.quitting`) is the primary integration surface for exporters, custom reporting, and setup/teardown.

**Web UI.** A Flask backend serves a React front-end that streams live throughput, response-time percentiles, and failures over polling; the same numbers are available as CSV export and via a small REST/stats API for automation.

## Production Notes

**One process = one core.** The most frequent scaling mistake is running a single Locust process and concluding "Locust can't generate load." A single process saturates one core well before it saturates most targets. Run distributed workers (`--worker`) equal to core count, and add machines horizontally; the master is lightweight and rarely the bottleneck.

**`HttpUser` vs `FastHttpUser`.** If the load generator's CPU is the limiter (visible as high generator CPU with the target under-loaded), switch users to `FastHttpUser`. It routinely yields several times the RPS per core. Watch for API differences: streaming, some auth helpers, and certain `requests` conveniences behave differently or are absent.

**Monkey-patch ordering.** gevent must patch the stdlib before other modules import socket-using libraries, or you get subtle blocking and "MonkeyPatchWarning" noise. Keep the `locustfile` import graph clean; import third-party network libraries lazily inside tasks if they misbehave.

**Percentiles are approximate.** Response-time percentiles are computed from rounded/bucketed histograms and aggregated across workers, so the reported p95/p99 are estimates, not exact order statistics. They are fine for trend and regression detection; do not treat them as precise SLA measurements to the millisecond.

**Client-side bottlenecks masquerade as server latency.** Because timings are measured in the generator, a saturated generator inflates reported response times. Always confirm generator CPU/network headroom before trusting a latency regression.

**Version pinning matters.** The pre-1.0 API (`HttpLocust`, `TaskSet`-centric locustfiles) and the modern `HttpUser` API are incompatible; copy-pasted examples from old blog posts will not run. Pin `locust` in CI and read the changelog before major upgrades — 1.0 and 2.0 both carried breaking changes[^3].

## When to Use / When Not

**Use when:**
- Your team already writes Python and wants tests that live in version control alongside code.
- Test logic needs real branching, computed data, shared state, or reuse of existing Python client libraries.
- You need distributed load with a live dashboard and CI-friendly headless runs from the same tool.
- The workload is I/O-bound HTTP(S) or a protocol you can wrap in a small client.

**Avoid when:**
- You need maximum RPS from minimal hardware for a simple constant-rate HTTP benchmark — a compiled tool (k6, wrk, vegeta) is more efficient per core.
- Your team doesn't write Python and wants GUI-authored or record-and-replay scenarios (JMeter fits better).
- Test logic is CPU-heavy per request (crypto, large payload transforms) — that fights the cooperative greenlet model.
- You require exact, unbucketed latency percentiles for contractual SLA reporting.

## Alternatives

- grafana/k6 — Go engine scripted in JavaScript; use instead when you want higher throughput per core and don't need Python.
- apache/jmeter — mature GUI/XML tool with broad protocol support; use when the team prefers point-and-click scenarios over code.
- gatling/gatling — Scala/Java DSL with strong HTML reports; use in JVM shops wanting detailed built-in reporting.
- tsenart/vegeta — Go CLI for constant-rate HTTP; use for simple, precise fixed-throughput benchmarks.
- wg/wrk — C single-URL benchmarking; use when you just need maximum requests against one endpoint.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-02 | First public release by Jonatan Heyman[^1]. |
| 0.x | 2011–2020 | `HttpLocust` / `TaskSet` era API. |
| 1.0 | 2020-05 | Major API rework: `User`/`HttpUser`, `@task` on the user class, dropped Python 2[^3]. |
| 2.0 | 2021-08 | Reworked user-distribution/dispatch across workers; further breaking changes[^3]. |
| 2.x | 2022–2026 | `FastHttpUser` maturation, `LoadTestShape`, modernized React web UI, ongoing releases. |

## References

[^1]: Locust project history and authors, README — https://github.com/locustio/locust
[^2]: GitHub API, `repos/locustio/locust` — ~27,993 stars, 3,221 forks, MIT, last push 2026-07-14 (fetched 2026-07-15).
[^3]: Locust changelog / release history — https://docs.locust.io/en/stable/changelog.html

## Tags

python, load-testing, performance-testing, gevent, http, benchmarking, distributed-systems, cli, testing-tools, greenlet
