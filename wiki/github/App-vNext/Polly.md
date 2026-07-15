# App-vNext/Polly

> .NET resilience and transient-fault-handling library — Retry, Circuit Breaker, Timeout, Rate Limiter, Hedging, and Fallback expressed as composable, thread-safe pipelines.

[GitHub repo](https://github.com/App-vNext/Polly) ·
[Official website](https://www.thepollyproject.org) ·
[License: BSD-3-Clause](https://github.com/App-vNext/Polly/blob/main/LICENSE)

## Overview

Polly is the de facto resilience library for .NET. It lets you wrap a delegate
(usually an outbound HTTP or database call) in one or more strategies — retry,
circuit breaker, timeout, rate limiter, hedging, fallback — and execute it with
transient-fault handling applied uniformly. The project dates to 2013, is a
member of the .NET Foundation, and is depended on transitively by a large share
of the .NET server ecosystem[^1]. With ~14k stars and single-digit open issues
against near-daily pushes, it is one of the better-maintained libraries in the
ecosystem; low issue count here reflects fast triage, not neglect.

The defining event in Polly's history is the **v8 rewrite** (2023), developed in
collaboration with Microsoft[^2]. v8 replaced the old `Policy` / `PolicyWrap`
object model with a `ResiliencePipeline` built from strategy options objects,
unified the synchronous and asynchronous code paths, and cut per-call
allocations substantially. The trade-off: the fluent API most existing tutorials
and Stack Overflow answers describe is the *legacy* v7 API, still shipped in the
`Polly` package but frozen. New code should target `Polly.Core`.

Polly is also the engine underneath Microsoft's own
`Microsoft.Extensions.Http.Resilience` (the standard resilience handler for
`HttpClient` and `IHttpClientFactory`)[^3]. Many teams consume Polly indirectly
through that package without ever calling the Polly API directly.

## Getting Started

```sh
dotnet add package Polly.Core
```

```cs
// A pipeline is an ordered composition of resilience strategies.
ResiliencePipeline pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 4,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,                       // decorrelated jitter
        Delay = TimeSpan.FromSeconds(1),
    })
    .AddTimeout(TimeSpan.FromSeconds(10))       // per-attempt outer timeout
    .Build();

await pipeline.ExecuteAsync(async token =>
{
    await CallDownstreamServiceAsync(token);
}, cancellationToken);
```

Strategy order matters: the first strategy added is the outermost. A retry placed
before a timeout retries the whole timed operation; the reverse times out each
individual retry.

## Architecture / How It Works

Strategies split into two families. **Reactive** strategies inspect the outcome
(exception or returned result) — Retry, Circuit Breaker, Fallback, Hedging.
**Proactive** strategies act without needing a failure — Timeout and Rate
Limiter, which cancel or reject executions preemptively.

The v8 core is built around a small set of primitives. A `ResiliencePipeline`
executes a callback through a linked chain of strategies. Each strategy is
configured with a strongly-typed options object (`RetryStrategyOptions`,
`CircuitBreakerStrategyOptions<T>`, etc.) whose predicates are expressed with
`PredicateBuilder`. Executions carry a `ResilienceContext` (a pooled, reusable
object holding the operation key, cancellation token, and arbitrary property bag)
which is what lets v8 avoid the closure allocations that plagued v7. This is why
the docs push `static` lambdas everywhere — the API is designed so the hot path
allocates nothing.

Circuit breakers expose a `CircuitBreakerStateProvider` (read the
Closed/Open/HalfOpen/Isolated state for health checks) and a
`CircuitBreakerManualControl` (isolate or close the circuit by hand, e.g. to shed
a known-bad downstream). Hedging — new in v8 — fires parallel attempts when the
primary is slow and takes the first to complete; it is the mechanism behind
tail-latency reduction and is distinct from retry (which is sequential and
failure-driven).

Packaging is deliberately layered: `Polly.Core` has zero third-party
dependencies; `Polly.Extensions` adds DI registration and telemetry (metrics and
logging via `Microsoft.Extensions`); `Polly.RateLimiting` bridges to the
framework's `System.Threading.RateLimiting`; `Polly.Testing` exposes pipeline
internals for assertions. Chaos-engineering strategies (formerly the separate
Simmy project) are now folded into `Polly.Core` as chaos strategies[^4].

## Production Notes

- **v7 vs v8 is a real fork in the road.** The legacy `Policy.Handle<T>().Retry()`
  API still works but is maintenance-only. Mixing v7 policies and v8 pipelines in
  one codebase is a common source of confusion; pick one per project.
- **Bulkhead is gone.** The v7 `Bulkhead` isolation policy has no v8 equivalent —
  use the rate limiter (specifically a concurrency limiter) via
  `Polly.RateLimiting` instead. This trips up teams porting v7 code.
- **Prefer the HttpClient handler for HTTP.** If your use case is outbound HTTP,
  `Microsoft.Extensions.Http.Resilience`'s `AddStandardResilienceHandler()` gives
  you a sensible pre-tuned pipeline (retry + circuit breaker + timeouts + rate
  limiter) with correct ordering, rather than hand-assembling one[^3].
- **Timeouts require cooperative cancellation.** The timeout strategy only works
  if your delegate honors the `CancellationToken` it is handed. A blocking call
  that ignores the token will run past the timeout; Polly cannot abort a thread.
- **Pipelines are reusable and thread-safe; build once.** Construct pipelines at
  startup (ideally via DI with `AddResiliencePipeline`) and reuse them. Rebuilding
  per request throws away the pooling and, for circuit breakers, resets the state
  you depend on.
- **Retry storms.** Naive retry without jitter across many clients produces
  synchronized retry waves against a recovering service. Always enable
  `UseJitter`, and layer a circuit breaker so retries stop when the downstream is
  clearly down.
- **Telemetry is opt-in.** Metrics and enriched logging live in
  `Polly.Extensions`; without it you get no built-in observability of retries or
  breaker transitions.

## When to Use / When Not

**Use when:**
- You are writing .NET services that call networks, databases, or queues and need
  retry / circuit-breaking / timeout without hand-rolling it.
- You want resilience configured declaratively and registered through DI.
- You need tail-latency mitigation (hedging) or health-aware circuit state.

**Avoid / reconsider when:**
- Your only need is standard HTTP retry — the `Microsoft.Extensions.Http.Resilience`
  standard handler (which wraps Polly) is less code.
- You are not on .NET — Polly is .NET-only; JVM and Go have their own libraries.
- You need distributed rate limiting or breaker state shared across instances —
  Polly's state is in-process only; coordinate externally (e.g. Redis) yourself.

## Alternatives

- dotnet/extensions — `Microsoft.Extensions.Http.Resilience`; the higher-level,
  HttpClient-focused resilience handler built *on top of* Polly. Use when you only
  need opinionated HTTP resilience with minimal setup.
- resilience4j/resilience4j — the JVM equivalent (retry, breaker, bulkhead, rate
  limiter). Use on Java/Kotlin services instead of Polly.
- failsafe-lib/failsafe — another JVM resilience library with a fluent policy
  model similar to Polly's. Use on the JVM when you prefer its API.
- slok/goresilience — Go resilience runners (circuit breaker, retry, timeout,
  bulkhead). Use for Go services.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2013–2015 | Original synchronous retry / circuit-breaker `Policy` API. |
| 5.0 | 2018 | Async support consolidated, `PolicyWrap` composition. |
| 7.0 | 2019 | `Context`, `PolicyRegistry`, generic `Policy<TResult>` maturity. |
| 8.0 | 2023-11 | Ground-up rewrite: `ResiliencePipeline`, options objects, low-allocation core, hedging, rate limiter, Microsoft collaboration[^2]. |
| 8.x | 2024–2026 | Chaos strategies (ex-Simmy) merged, telemetry and API refinements[^4]. |

## References

[^1]: Polly project — home and .NET Foundation membership. https://www.thepollyproject.org
[^2]: Polly v8 announcement and rationale (rewrite, Microsoft collaboration). https://www.pollydocs.org/migration-v8
[^3]: Microsoft Learn, "Build resilient HTTP apps: Microsoft.Extensions.Http.Resilience". https://learn.microsoft.com/dotnet/core/resilience/http-resilience
[^4]: Polly documentation — chaos engineering strategies. https://www.pollydocs.org/chaos/

## Tags

csharp, dotnet, resilience, retry, circuit-breaker, fault-tolerance, rate-limiting, hedging, transient-fault-handling, library
