# Netflix/concurrency-limits

> A Java library that borrows TCP congestion-control math to auto-discover a service's safe concurrency limit at runtime, instead of hand-tuning static RPS caps.

[GitHub repo](https://github.com/Netflix/concurrency-limits) ·
[License: Apache-2.0](https://github.com/Netflix/concurrency-limits/blob/main/LICENSE)

## Overview

`concurrency-limits` implements adaptive load-shedding: rather than configuring a fixed requests-per-second ceiling, each node measures its own request latency and continuously re-estimates how many concurrent in-flight requests it can serve before a queue builds. The premise, laid out in the README and Netflix's 2018 "Performance Under Load" writeup, is that static RPS limits go stale the moment a service auto-scales or its dependencies slow down, and a stale limit either wastes capacity or fails to protect the node when it matters[^1][^2].

The core idea is to treat a service's concurrency limit like a TCP congestion window. Little's Law (`Limit ≈ average RPS × average latency`) says the two views are equivalent, so the library reuses delay-based congestion algorithms — chiefly variants of TCP Vegas — to grow the limit while latency is flat and shrink it aggressively when latency rises or requests are dropped[^1]. The defining tension is that this is an estimator, not a guarantee: it reacts to latency signal, so it protects against gradual queue buildup well but can misread noisy latency (GC pauses, neighbor tenants, bursty traffic) and either clamp throughput too hard or react a beat too late.

The library is Java-first and predates most of the current "adaptive concurrency" features now built into service meshes (Envoy adaptive concurrency filter draws on the same ideas). It is widely referenced as a reference implementation even by teams who do not adopt it directly.

## Getting Started

Published to Maven Central under `com.netflix.concurrency-limits`. The core artifact carries the limit algorithms; integration artifacts (`concurrency-limits-grpc`, `concurrency-limits-servlet`, etc.) add interceptors[^3].

```gradle
dependencies {
    implementation 'com.netflix.concurrency-limits:concurrency-limits-core:0.4.5'
    implementation 'com.netflix.concurrency-limits:concurrency-limits-grpc:0.4.5'
}
```

```java
// A limiter you drive by hand: acquire, then report the outcome.
Limiter<Void> limiter = SimpleLimiter.newBuilder()
    .limit(Gradient2Limit.newBuilder().build())
    .build();

Optional<Limiter.Listener> listener = limiter.acquire(null);
if (!listener.isPresent()) {
    // over limit — shed load (return 429 / UNAVAILABLE / fail fast)
    return reject();
}
try {
    doWork();
    listener.get().onSuccess();   // sampled into the limit estimate
} catch (Exception e) {
    listener.get().onDropped();   // signals overload — limit backs off
    throw e;
} 
```

The `acquire` → `onSuccess`/`onIgnore`/`onDropped` contract is the whole API surface; every integration is a thin adapter around it.

## Architecture / How It Works

Three concepts compose: a **Limit** algorithm (how big should the window be?), a **Limiter** (enforcement + partitioning), and an integration adapter.

**Limit algorithms** live in `com.netflix.concurrency.limits.limit`:

- **VegasLimit** — delay-based. Estimates queue size as `L·(1 − minRTT/sampleRTT)`; each sampling window it adds 1 to the limit if the estimated queue is below `alpha` (~2–3) or subtracts 1 if above `beta` (~4–6)[^1].
- **Gradient2Limit** — the recommended default. Tracks the divergence between a short and a long exponential moving average of latency, which cancels out the bias and drift that plague `minRTT`-anchored estimators when the true minimum latency itself drifts. Sustained divergence is read as a queueing trend and the limit is cut hard[^1].
- **AIMDLimit** — pure loss-based (additive-increase/multiplicative-decrease), reacting to drops/timeouts rather than latency. Recommended on the client side.
- **WindowedLimit** — a wrapper that batches samples over a time window before feeding the inner algorithm, smoothing bursty traffic.
- Plus `SettableLimit`, `FixedLimit` for testing and static fallback.

**Limiters** wrap a Limit with enforcement policy. `SimpleLimiter` counts inflight requests against one gauge and rejects at the ceiling. Partitioned limiters carve the window into named slices — e.g. `partition("live", 0.9).partition("batch", 0.1)` guarantees live traffic 90% and lets batch use only headroom, so a retry storm on batch cannot starve live requests. Partition keys come from a gRPC header, a servlet `Principal`, or any lookup function.

**Integrations** are the outer layer: `ConcurrencyLimitServerInterceptor` / `ConcurrencyLimitClientInterceptor` for gRPC, `ConcurrencyLimitServletFilter` (rejects with HTTP 429), and `BlockingAdaptiveExecutor`, which resizes a thread pool to track the measured limit and blocks callers at the ceiling. The library recommends a delay-based server limiter (Vegas/Gradient2) paired with a loss-based or blocking client limiter for backpressure.

## Production Notes

- **It sheds load; it does not queue politely.** Once the estimated limit is hit, requests are rejected immediately (`UNAVAILABLE` / 429) unless you opt into a blocking limiter. Callers must be ready to handle fast rejection — this is a circuit-breaker-adjacent behavior, not a rate smoother.
- **Latency signal quality is everything.** The algorithms infer queueing from latency. Anything that adds latency noise unrelated to load — stop-the-world GC, noisy-neighbor CPU steal, downstream jitter — can be misread as queue buildup and clamp the limit below true capacity. Gradient2 exists specifically to reduce this, but it cannot fully eliminate it. Validate under realistic load before trusting it in the critical path.
- **Very slow, uniform requests fool it.** Little's Law ties limit to latency; a workload where every request is legitimately slow yields a small limit even when the box has headroom. Tune `minLimit`/`maxLimit` bounds to bracket the estimator.
- **Per-node, not global.** Each instance estimates independently. There is no shared/global limit coordination; behavior across a fleet is emergent. This is by design (no coordination point) but means you cannot reason about a cluster-wide ceiling from config alone.
- **Maintenance is slow.** The project is production-proven at Netflix but has low external commit velocity and has never shipped a 1.0 — it has lived in the 0.x range for its whole life. Treat the API as stable-by-inertia rather than actively evolving, and pin your version. Newer greenfield systems increasingly get equivalent behavior from the service mesh (Envoy's adaptive concurrency filter) instead of an in-process library.
- **Metrics matter more than usual.** Because the limit moves continuously, you are effectively blind without exporting the current limit, inflight count, and drop rate. Wire the `MetricRegistry` hooks into your telemetry before rollout, not after an incident.

## When to Use / When Not

**Use when:**
- You run JVM services (gRPC or servlet) and want load-shedding that adapts as you auto-scale or as dependencies slow, without re-tuning static RPS caps.
- You need per-request-class QoS (protect live traffic from batch) via partitioned limits.
- You want client-side backpressure so a batch job cannot overwhelm a shared dependency.

**Avoid when:**
- You are not on the JVM — there is no first-class port; the value is in the Java integrations.
- You already run a service mesh with adaptive concurrency (Envoy/Istio) — you would be duplicating the mechanism in-process.
- Your workload has inherently noisy or bimodal latency that will confuse a latency-based estimator; a well-load-tested static limit may be more predictable.
- You need a hard, globally-coordinated ceiling (billing quotas, contractual limits) — this is a local estimator, not an enforcement authority.

## Alternatives

- resilience4j/resilience4j — JVM bulkhead + rate limiter + circuit breaker; use when you want explicit, statically-configured limits and richer fault-tolerance primitives rather than an adaptive estimator.
- envoyproxy/envoy — adaptive concurrency filter implements the same gradient idea at the proxy layer; use when you want it out of process and language-agnostic across a mesh.
- App-Vision/... token-bucket libraries (e.g. Guava `RateLimiter`, Bucket4j) — use when a fixed RPS cap is genuinely what you want and simplicity beats adaptivity.
- alibaba/Sentinel — flow control / load-shedding with adaptive "system load" rules; use when you want a broader traffic-governance framework with a dashboard, common in the Spring Cloud Alibaba stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-12 | Repo opened; core Vegas + gRPC interceptors[^3]. |
| —       | 2018   | Netflix Tech Blog "Performance Under Load" popularizes the approach; Gradient/Gradient2 algorithms added[^2]. |
| 0.x     | ongoing | Remained pre-1.0 throughout; servlet filter, executor, and partitioned limiters landed across 0.3–0.4 releases. |

(Exact per-release version numbers and dates are not restated here to avoid error — check Maven Central / GitHub Releases for the current pinned version[^3].)

## References

[^1]: Netflix/concurrency-limits README — algorithm descriptions (Vegas, Gradient2), enforcement and integration examples. https://github.com/Netflix/concurrency-limits
[^2]: Netflix Technology Blog, "Performance Under Load" — the design rationale for adaptive concurrency limits. https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581
[^3]: Maven Central — `com.netflix.concurrency-limits` artifacts (core, grpc, servlet, spectator). https://central.sonatype.com/search?q=g:com.netflix.concurrency-limits

## Tags

java, jvm, load-shedding, adaptive-concurrency, congestion-control, backpressure, grpc, rate-limiting, resilience, netflix-oss, little-law
