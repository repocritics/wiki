# micrometer-metrics/micrometer

> A vendor-neutral facade for application metrics and observability — SLF4J, but for your monitoring backend.

[GitHub repo](https://github.com/micrometer-metrics/micrometer) ·
[Official website](https://micrometer.io) ·
[License: Apache-2.0](https://github.com/micrometer-metrics/micrometer/blob/main/LICENSE)

## Overview

Micrometer is a Java instrumentation library that decouples your code from the monitoring system it reports to. You instrument once against Micrometer's API — counters, gauges, timers, distribution summaries — and choose the backend (Prometheus, Datadog, New Relic, CloudWatch, Graphite, InfluxDB, OTLP, and a dozen others) via a registry dependency, ideally at deploy time rather than compile time[^1]. Its metrics are *dimensional*: every measurement carries key/value tags, so a single `http.server.requests` timer fans out by method, URI, and status.

Micrometer's reach comes almost entirely from Spring Boot, which adopted it as the metrics engine in Boot 2.0 (2018) and wires it into Actuator by default[^2]. For most Java teams, Micrometer is not a library they chose but the one that came with the framework. It is usable standalone, and the maintainers (originally led by Jon Schneider, drawing on Netflix's Atlas/Spectator lineage) treat framework-independence as a design goal, but the gravitational center is the Spring ecosystem.

The defining tension is the facade abstraction itself. Vendor-neutral naming means Micrometer owns the metric name and lets each registry translate it to backend conventions (dots for Micrometer, underscores for Prometheus, camelCase for others). That indirection is the whole value proposition — and the source of most surprises, because the name you write is not the name you query.

## Getting Started

Gradle, with a Prometheus backend:

```groovy
dependencies {
    implementation 'io.micrometer:micrometer-core'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

```java
PrometheusMeterRegistry registry =
    new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);

Counter orders = Counter.builder("orders.placed")
    .description("Total orders placed")
    .tag("region", "kr")
    .register(registry);

orders.increment();

Timer.builder("db.query")
    .publishPercentileHistogram()          // server-aggregatable buckets
    .register(registry)
    .record(() -> repository.findAll());

// expose registry.scrape() at GET /metrics for Prometheus to pull
```

Under Spring Boot, none of this is manual: adding `micrometer-registry-prometheus` to the classpath auto-configures the registry and the `/actuator/prometheus` endpoint.

## Architecture / How It Works

The core abstraction is `MeterRegistry`. A `Meter` (Counter, Gauge, Timer, DistributionSummary, LongTaskTimer, FunctionCounter, FunctionTimer) is created against a registry and identified by its name plus its full set of tags — two timers with the same name but different tags are different meters. Concrete registries subclass `MeterRegistry` and implement the transport to one backend.

Registries fall into two families:

- **Pull-based** — `PrometheusMeterRegistry` holds all state in memory and renders it on demand when Prometheus scrapes. Nothing is pushed; missed scrapes are gaps, not lost counts.
- **Push-based (`StepMeterRegistry`)** — Datadog, New Relic, InfluxDB, StatsD, CloudWatch, and most SaaS registries. They aggregate over a fixed *step* interval and ship a batch. Counters and rates reset each step; a failed publish drops that interval's data permanently.

A `CompositeMeterRegistry` fans one instrumentation call out to several backends at once, and `Metrics.globalRegistry` is a static composite for code that can't inject a registry.

Two cross-cutting mechanisms matter. **`MeterFilter`** runs at registration time to deny, rename, re-tag, or cap the cardinality of meters — the primary defense against tag explosions and the hook for enforcing naming conventions. **Percentiles** come in two incompatible flavors: client-side percentiles (`publishPercentiles`) are computed per-instance and *cannot be aggregated* across a fleet, whereas histogram buckets (`publishPercentileHistogram`) ship raw buckets that the backend aggregates. Choosing the former for a multi-instance service is a recurring mistake.

Since 1.10 (2022) Micrometer also ships the **Observation API** and **Micrometer Tracing**[^3]. An `Observation` is instrumented once and can emit metrics *and* start a trace span simultaneously; Micrometer Tracing is itself a facade over OpenTelemetry and OpenZipkin Brave, mirroring the metrics story for distributed tracing. This is the project's expansion from "metrics facade" to "observability facade," and it is why the repo description now says observability.

## Production Notes

**Tag cardinality is the number-one operational hazard.** Every distinct tag-value combination is a separate meter held in memory and a separate time series billed by your backend. Putting a user ID, raw URL with path parameters, or unbounded error message into a tag will exhaust heap and generate five- or six-figure metric bills. Bound cardinality at the source and add a `MeterFilter.maximumAllowableTags` backstop.

**Gauges hold a weak reference.** `Gauge.builder("cache.size", cache, Cache::size)` does not keep `cache` alive. If the referenced object is garbage-collected — very common when a gauge is registered against a short-lived local — the gauge silently reports `NaN` forever. Register gauges against long-lived objects and keep a strong reference.

**Histograms multiply time series.** `publishPercentileHistogram` on a timer can emit dozens of bucket series per tag combination. Combined with cardinality this compounds fast; scope histograms to the handful of metrics where latency distribution actually matters.

**Names are not what you write.** `http.server.requests` becomes `http_server_requests_seconds` in Prometheus and something else in Datadog. Query against the *backend's* rendered name, not the Java string. Let the registry do the translation; do not hand-format names for one backend, since that breaks the moment you add a second.

**Step registries lose the last interval on shutdown.** A JVM that exits mid-step drops the un-published window. For batch jobs and serverless, call `close()`/`shutdown()` to force a final publish, or the run's tail metrics vanish.

**Version is managed by Spring Boot.** Boot's dependency management pins the Micrometer version; overriding it independently can desync `micrometer-core`, the registry modules, and Micrometer Tracing, which must all move together. Prefer the BOM. Micrometer runs on Java 8+, with a few modules (micrometer-java11, micrometer-jetty11) requiring newer JDKs.

## When to Use / When Not

**Use when:**
- You're on Spring Boot — it's already there and integrated.
- You want to defer or change the monitoring backend without re-instrumenting.
- You need dimensional metrics with a clean Java API and don't want backend-specific client code scattered through the app.
- You want metrics and tracing from a single instrumentation point via the Observation API.

**Avoid when:**
- You want vendor-neutral metrics *and* traces *and* logs under one spec across many languages — OpenTelemetry is the broader standard (Micrometer bridges to it but is Java-only).
- You only ever target Prometheus and want zero abstraction — the direct Prometheus Java client is simpler.
- You're not on the JVM at all.
- You need push-frequency or exactly-once delivery guarantees the step-registry model doesn't provide.

## Alternatives

- open-telemetry/opentelemetry-java — the vendor-neutral CNCF standard covering metrics, traces, and logs across languages; use instead when you want one cross-language spec and Micrometer's Java-only, metrics-first scope is too narrow.
- prometheus/client_java — direct Prometheus instrumentation; use when Prometheus is the only backend you will ever have and the facade is pure overhead.
- dropwizard/metrics — the older, hierarchical (non-dimensional) Java metrics library; use only for legacy systems already built on it, since tag-based backends fit poorly.
- Netflix/spectator — Micrometer's Atlas-focused predecessor; use when you're inside Netflix's Atlas stack, otherwise Micrometer supersedes it.
- micrometer-metrics/tracing — same project's tracing facade over OpenTelemetry and Brave; pair with, don't replace, the metrics core when you also need spans.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2018-02 | First GA, shipped as the metrics engine of Spring Boot 2.0[^2]. |
| 1.5.0 | 2020-05 | Expanded registry set; base-unit and naming refinements. |
| 1.9.0 | 2022-07 | Last line before the Observation API. |
| 1.10.0 | 2022-11 | Observation API + Micrometer Tracing 1.0 (absorbing Spring Cloud Sleuth's role)[^3]. |
| 1.12.0 | 2023-11 | OTLP registry maturation, further Observation adoption. |
| 1.15.x | 2026 | Milestone releases now published to Maven Central[^4]. |

## References

[^1]: Micrometer, "Concepts / Registry" documentation. https://docs.micrometer.io/micrometer/reference/concepts.html
[^2]: Spring Boot 2.0 release notes — Micrometer as the metrics backend. https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Release-Notes
[^3]: Micrometer blog, "Micrometer 1.10.0 released" — Observation API and Micrometer Tracing. https://micrometer.io/blog/
[^4]: micrometer-metrics/micrometer README — milestone releases published to Maven Central starting 1.15.0-M2. https://github.com/micrometer-metrics/micrometer

## Tags

java, jvm, metrics, observability, monitoring, prometheus, spring-boot, facade, dimensional-metrics, distributed-tracing, cloud-native
