# open-telemetry/opentelemetry-python

> The vendor-neutral OpenTelemetry API and SDK for Python — traces, metrics, and logs behind one instrumentation surface.

[GitHub repo](https://github.com/open-telemetry/opentelemetry-python) ·
[Official website](https://opentelemetry.io) ·
[License: Apache-2.0](https://github.com/open-telemetry/opentelemetry-python/blob/main/LICENSE)

## Overview

OpenTelemetry (OTel) is the CNCF observability standard formed in 2019 by merging the OpenTracing and OpenCensus projects[^1]. This repository is the Python core: the `opentelemetry-api` package (abstract classes plus no-op implementations that follow the language-agnostic OpenTelemetry specification), the `opentelemetry-sdk` reference implementation, and `opentelemetry-semantic-conventions`. It is the second-most-starred OTel language SIG after Go and is actively maintained, with commits landing daily as of mid-2026.

The defining design decision is the **API/SDK split**. Instrumented libraries are expected to depend only on `opentelemetry-api`, which does nothing on its own — no exporters, no background threads, no cost — until an application installs and configures the SDK. This lets a library emit telemetry without forcing an observability backend on its users, and lets the application own the entire pipeline choice. It is the same contract as SLF4J in Java: a facade plus a pluggable implementation.

The tension is maturity-by-signal. Traces and metrics are marked Stable and covered by the project's 1.x compatibility guarantees; the Logs signal is still in Development as of 2026 and ships breaking changes[^2]. Anyone adopting OTel logging today is tracking a moving target, while tracing is safe to build on. The three signals live in one repo but do not share one stability story.

## Getting Started

```sh
pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp
```

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Wire the SDK once, at application startup — not in a library.
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("handle_request") as span:
    span.set_attribute("http.route", "/checkout")
    # ... work happens inside the active span ...
```

Zero-code auto-instrumentation is available via the contrib repository:

```sh
pip install opentelemetry-distro opentelemetry-instrumentation
opentelemetry-bootstrap -a install     # detect installed libs, add instrumentors
opentelemetry-instrument python app.py # inject tracing/metrics at import time
```

## Architecture / How It Works

The stack is three layers, each independently installable:

1. **API** (`opentelemetry-api`) — `TracerProvider`, `MeterProvider`, `LoggerProvider`, plus context and propagation primitives. Ships no-op implementations so unconfigured code stays cheap. This is the only thing libraries should import.
2. **SDK** (`opentelemetry-sdk`) — the reference implementation: span processors, samplers, `Resource` detection, metric readers/views, and the log-record pipeline. Configured by the application, largely through `OTEL_*` environment variables.
3. **Exporters / propagators** — separate packages (`opentelemetry-exporter-otlp-proto-grpc`, `-http`, console, etc.). OTLP is the canonical wire format; the older Jaeger and Zipkin exporters were deprecated in favor of sending OTLP to a Collector.

**Context propagation** rides on Python's `contextvars`. This means the active span follows `asyncio` tasks and coroutines correctly with no extra work, but does **not** automatically cross `threading.Thread` or `concurrent.futures` boundaries — worker threads start with an empty context unless you capture and `context.attach()` it yourself. This is the single most common source of "my spans are orphaned / parented wrong" confusion.

**Span processors** decide how spans leave the process. `SimpleSpanProcessor` exports synchronously (test/debug only); `BatchSpanProcessor` buffers in a bounded queue and flushes from a background thread — the production default. Sampling is **head-based** and parent-based by default: the decision is made when the span starts and propagated downstream. Tail-based sampling (decide after seeing the whole trace) is not possible in-process and requires the separate OpenTelemetry Collector.

Metrics use an explicit temporality model (cumulative vs. delta) and a `View` API to rename, filter, or re-aggregate instruments before export. The exporter's temporality must match what the backend expects, which is a frequent misconfiguration.

## Production Notes

**The dual version scheme is a real trap.** Stable packages (`api`, `sdk`, semantic conventions core) are versioned `1.x.y`; experimental packages historically carried a parallel `0.Nbeta` number, and releases pin them together (e.g. an `1.x` release ships alongside a `0.xxb0`). Copy-pasting a single version pin across all OTel packages will produce unresolvable dependency sets. Pin from a known-good release's published matrix, not by guessing.

**Fork safety.** The `BatchSpanProcessor` export thread does not survive `os.fork()`. Under Gunicorn/uWSGI with preloaded apps, the provider is initialized in the master and the export thread is gone in the workers, so telemetry silently stops. Initialize (or re-initialize) the SDK in a `post_fork` / worker-boot hook, not at module import time.

**Silent drops.** `BatchSpanProcessor` has a bounded queue (default 2048). Under burst load it drops spans without raising — you get gaps, not errors. Tune `max_queue_size`, `max_export_batch_size`, and `schedule_delay_millis`, and watch for the dropped-span log warnings.

**Instrumentation lives elsewhere.** The library instrumentors (Flask, Django, requests, psycopg, boto, etc.) are in the sibling `opentelemetry-python-contrib` repo and release on a coupled but separate cadence. Version skew between core and contrib is a common breakage after upgrades; upgrade them together.

**Overhead is not free.** Per-span attribute setting, context attach/detach, and exporter serialization add measurable latency in hot paths. High-throughput services should sample aggressively (head sampling here, or ship 100% to a Collector doing tail sampling) rather than exporting every span.

**Logs are pre-stable.** The logging bridge and `LoggerProvider` API can and do change between minor releases. Treat production log-signal adoption as opt-in-with-churn, and keep the code that wires it isolated so breaking changes are localized.

## When to Use / When Not

**Use when:**
- You want vendor-neutral instrumentation and the freedom to switch or fan out to multiple backends (Jaeger, Prometheus, Datadog, Honeycomb, Grafana) without rewriting app code.
- You are authoring a library and want to emit telemetry without imposing an SDK or backend on downstream users.
- You need traces and metrics under a real compatibility guarantee and are standardizing across a polyglot fleet on one spec.

**Avoid when:**
- You want a batteries-included APM with error grouping, dashboards, and alerting out of the box — OTel is the pipe, not the destination; you still need a backend.
- Your only need is basic error tracking; a purpose-built error monitor is lighter.
- You need stable structured logging today — the Logs signal is still in development.
- You are on an unsupported Python; the project drops end-of-life Pythons roughly six months after upstream EOL and requires a modern 3.x.

## Alternatives

- open-telemetry/opentelemetry-python-contrib — sibling repo, not a competitor: the auto-instrumentation and library integrations. Use it alongside core, not instead of it.
- open-telemetry/opentelemetry-collector — run it when you need tail sampling, buffering, or backend fan-out outside the process.
- DataDog/dd-trace-py — use instead when you are all-in on Datadog and want their tuned tracer plus APM UI as one product.
- getsentry/sentry-python — use instead when error tracking and release health matter more than open tracing/metrics standards.
- census-instrumentation/opencensus-python — the deprecated predecessor; use only for legacy code mid-migration to OTel.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-05 | Repo created; OpenTracing + OpenCensus merge into OpenTelemetry[^1]. |
| 1.0.0 | 2021-02 | Tracing API/SDK declared stable[^3]. |
| 1.x / 0.x | 2021–2022 | Dual versioning: stable trace packages 1.x, experimental metrics/logs 0.x. |
| 1.15.0 | 2023-01 | Metrics API/SDK reach stability[^2]. |
| 1.x | 2024–2026 | Logs signal in active development; semantic conventions stabilizing; Python 3.10+ baseline. |

## References

[^1]: CNCF, "OpenTracing and OpenCensus merge to form OpenTelemetry" — 2019. https://www.cncf.io/blog/2019/05/21/a-brief-history-of-opentelemetry-so-far/
[^2]: OpenTelemetry Python status and releases (per-signal stability). https://opentelemetry.io/docs/languages/python/
[^3]: OpenTelemetry Python releases. https://github.com/open-telemetry/opentelemetry-python/releases

## Tags

python, observability, distributed-tracing, metrics, logging, opentelemetry, telemetry, sdk, cncf, apache-2.0
