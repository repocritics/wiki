# prometheus/client_golang

> The reference Prometheus instrumentation library for Go — metric types, a registry, and an HTTP exposition handler.

[GitHub repo](https://github.com/prometheus/client_golang) ·
[Docs](https://pkg.go.dev/github.com/prometheus/client_golang) ·
[License: Apache-2.0](https://github.com/prometheus/client_golang/blob/main/LICENSE)

## Overview

client_golang is the canonical Go client for [Prometheus](https://prometheus.io).
It splits into two largely independent parts: the `prometheus` package, used to
instrument application code, and the `api` package, used to query a Prometheus
server over its HTTP API[^1]. The instrumentation half is what almost everyone
uses; the API-query half is still marked experimental and can break outside the
normal semver major-version cadence[^2].

The library defines four metric types — Counter, Gauge, Histogram, and Summary —
plus their labelled `*Vec` variants, registers them into a collector registry,
and exposes them in the Prometheus text (and protobuf) exposition format over an
`http.Handler`. Prometheus itself and most of the CNCF ecosystem use it, which
makes its conventions de facto standards even for people who never run Prometheus.

The defining tension is that the library is deliberately low-level and
unopinionated about cardinality. It will happily create an unbounded number of
time series from user-controlled label values, and the cost of that mistake lands
on the Prometheus server that scrapes you, not on the process emitting the
metrics. Most production pain here is really cardinality-management pain.

## Getting Started

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
```

```go
import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// Label set must stay small and bounded — see Production Notes.
var httpRequests = promauto.NewCounterVec(
	prometheus.CounterOpts{Name: "http_requests_total", Help: "Total HTTP requests."},
	[]string{"method", "status"},
)

func main() {
	httpRequests.WithLabelValues("GET", "200").Inc()
	http.Handle("/metrics", promhttp.Handler())
	http.ListenAndServe(":2112", nil)
}
```

`promauto` registers into the default registry at construction; the older
`NewCounterVec` + `MustRegister` pattern does the same in two steps. Either way,
registering two metrics with the same name panics.

## Architecture / How It Works

The core objects are **Collectors** and a **Registry**. A metric (Counter, Gauge,
etc.) is itself a Collector; a `*Vec` is a Collector that manages a family of
child metrics keyed by label values. When the `/metrics` endpoint is scraped, the
registry calls `Collect` on every registered collector, gathers the resulting
samples, and serializes them.

- **Metric types.** Counter (monotonic), Gauge (up/down), Histogram (bucketed,
  aggregatable server-side), Summary (client-computed quantiles over a sliding
  window). Choosing Histogram vs Summary is the most consequential decision a
  user makes — see Production Notes.
- **`*Vec` and labels.** `WithLabelValues(...)` lazily creates and caches a child
  series per unique label-tuple. There is no eviction: every distinct tuple lives
  for the process lifetime unless you call `DeleteLabelValues`. This is where
  cardinality explosions come from.
- **Registries.** A package-global default registry (`prometheus.DefaultRegisterer`)
  is pre-populated with Go-runtime and process collectors; `prometheus.NewRegistry()`
  builds isolated ones for tests or per-subsystem scoping.
- **`promhttp`.** Wraps a registry into an `http.Handler` with text/protobuf
  content negotiation, gzip, and concurrency limits, plus `InstrumentHandler*`
  middleware. **`promauto`** merges construct+register; **`testutil`**
  (`CollectAndCompare`, `ToFloat64`) asserts metric values without scraping.

The registry, exposition format, and metric types are stable and semver-governed;
the `api/` query client and anything tagged **EXPERIMENTAL** in the changelog are
explicitly exempt[^2]. Native histograms (a sparse, high-resolution encoding) are
the most significant recent addition and remain experimental on both sides[^3].

## Production Notes

**Cardinality is the footgun.** Each unique combination of label values on a
`*Vec` is a distinct time series stored forever on the Prometheus server. Putting
user IDs, raw request paths, or full URLs into a label silently generates millions
of series and can OOM the scraping Prometheus. Labels must be low-cardinality and
bounded (HTTP method, status class, route *template* not raw path). There is no
client-side guardrail — the library will not warn you.

**Summary vs Histogram.** Summaries compute quantiles (e.g. p99) inside the
client over a sliding window; those quantiles **cannot be aggregated** across
instances — averaging p99 across ten pods is meaningless. Histograms store bucket
counts, which *do* aggregate, letting `histogram_quantile()` run server-side over
any grouping. Prefer Histograms unless you need a single-instance quantile and
cannot pick buckets in advance. Summaries also cost more CPU per observation.

**Histogram buckets are effectively permanent.** Boundaries are fixed at
construction; if they don't bracket the real distribution the quantile estimates
are garbage, and changing them later relabels the series. `prometheus.DefBuckets`
targets sub-second web latencies — set buckets explicitly for anything else.
Native histograms remove the manual-bucket problem but are experimental and need
compatible Prometheus/tooling[^3].

**The default registry ships extra metrics.** `promhttp.Handler()` also exposes
Go runtime (`go_*`) and process (`process_*`) metrics. Usually wanted, but a
"minimal" endpoint isn't minimal — use `promhttp.HandlerFor(reg, ...)` with a
custom registry for control.

**`MustRegister` panics** on duplicate names or double registration, at startup.
In tests and plugin-style code with fuzzy init order, prefer explicit `Register`
and handle `AlreadyRegisteredError`. Every scrape also serializes every series,
so high-cardinality endpoints make `/metrics` large and slow; `promhttp`'s
concurrency-limiting options exist because unbounded concurrent scrapes of a big
registry are a real problem.

**Go support is narrow** — only the two most recent Go major releases; older
toolchains may compile but are unsupported[^4].

## When to Use / When Not

**Use when:**
- You run a Go service and scrape it with Prometheus (or any OpenMetrics-compatible
  system).
- You want the reference implementation whose exposition format and conventions
  everything else follows.
- You need direct, low-overhead counters/gauges/histograms without a metrics SDK
  abstraction layer.

**Avoid / reconsider when:**
- You want vendor-neutral telemetry across tracing, metrics, and logs — OpenTelemetry's
  Go SDK is the better fit and can still export to Prometheus.
- You need push-based delivery to a hosted backend (StatsD/DogStatsD-style); the
  Prometheus model is pull/scrape, and the Pushgateway is a narrow escape hatch,
  not a general push path.
- You are extremely allocation-sensitive on a hot path and your label set is
  dynamic — the `*Vec` lookup and map machinery has measurable overhead; lighter
  libraries exist.

## Alternatives

- VictoriaMetrics/metrics — much smaller, faster Prometheus-compatible instrumentation library; use when you want lower overhead and fewer features.
- open-telemetry/opentelemetry-go — use when you want one vendor-neutral SDK for metrics, traces, and logs, exporting to Prometheus or anything else.
- uber-go/tally — use when you need a multi-backend metrics facade (Prometheus, StatsD, M3) behind one API.
- DataDog/datadog-go — use when your backend is Datadog and you want push-based DogStatsD instead of scraping.
- prometheus/client_python, prometheus/client_java — the same project's clients for other languages; use for non-Go services.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-01 | Repository created; early instrumentation library[^5]. |
| 0.9.x | 2018 | Pre-1.0 line; `promauto` and registry refactors land. |
| 1.0.0 | 2019 | API stabilization; semver commitment for the stable packages[^6]. |
| 1.11–1.14 | 2021–2022 | `promhttp` and runtime-metrics improvements, Go module hygiene. |
| 1.16 | 2023 | Experimental native (sparse) histogram support[^3]. |
| 1.x (current) | 2026 | Actively maintained; v2 batched in a milestone, plus a `prometheus.V2.*` experiment in the 1.x line[^2]. |

## References

[^1]: README, "Prometheus Go client library." https://github.com/prometheus/client_golang
[^2]: README, "Important note about releases and stability" (API client experimental; v2 milestone; `prometheus.V2.*` experiment). https://github.com/prometheus/client_golang#important-note-about-releases-and-stability
[^3]: Prometheus blog, "Native Histograms." https://prometheus.io/docs/concepts/metric_types/#histogram
[^4]: README, "Version Compatibility" (two most recent Go major releases). https://github.com/prometheus/client_golang/blob/main/RELEASE.md
[^5]: GitHub repository metadata, created 2013-01-25. https://github.com/prometheus/client_golang
[^6]: Prometheus documentation, "Instrumenting a Go application." https://prometheus.io/docs/guides/go-application/

## Tags

go, prometheus, metrics, observability, monitoring, instrumentation, time-series, http-handler, cncf, telemetry
