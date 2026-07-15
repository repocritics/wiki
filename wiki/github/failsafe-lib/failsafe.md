# failsafe-lib/failsafe

> Zero-dependency fault-tolerance library for the JVM — retries, circuit breakers, rate limiters, timeouts, bulkheads, and fallbacks composed as policies around any callable.

[GitHub repo](https://github.com/failsafe-lib/failsafe) ·
[Official website](https://failsafe.dev) ·
[License: Apache-2.0](https://github.com/failsafe-lib/failsafe/blob/master/LICENSE)

## Overview

Failsafe is a Java 8+ library for wrapping fallible logic in one or more resilience policies. It occupies a narrow niche: it does one thing — execute a callable through a composed chain of failure-handling policies — and carries no transitive dependencies, which is its main selling point over the heavier alternatives. It was created and is primarily maintained by Jonathan Halterman[^1].

The programming model is deliberately small. You describe *what counts as failure* and *how to react* declaratively (a `RetryPolicy`, a `CircuitBreaker`, a `Timeout`, etc.), then run your logic through `Failsafe.with(...).get(() -> ...)`. The same policy objects work for synchronous calls and for asynchronous execution returning a `CompletableFuture`. There is no annotation processor, no Spring dependency, no reactive-stack requirement in the core — that minimalism is the defining tradeoff. Teams that want batteries-included metrics, Spring Boot auto-configuration, and Micrometer wiring generally reach for Resilience4j instead; teams that want a single small jar and full programmatic control reach for Failsafe.

The most consequential thing to understand about Failsafe is **policy statefulness and composition order** (below). Both are easy to get subtly wrong in ways that compile and pass a happy-path test but silently defeat the resilience you thought you had.

## Getting Started

Maven — note the coordinates changed at v3 from `net.jodah:failsafe` to `dev.failsafe:failsafe`[^2]:

```xml
<dependency>
  <groupId>dev.failsafe</groupId>
  <artifactId>failsafe</artifactId>
  <version>3.3.2</version>
</dependency>
```

```java
import dev.failsafe.*;
import java.time.Duration;

RetryPolicy<Object> retry = RetryPolicy.builder()
    .handle(ConnectException.class)
    .withBackoff(Duration.ofMillis(100), Duration.ofSeconds(2))
    .withMaxRetries(3)
    .build();

// Synchronous
String body = Failsafe.with(retry).get(() -> httpGet("https://api.example.com"));

// Asynchronous — returns CompletableFuture<String>
Failsafe.with(retry).getAsync(() -> httpGet("https://api.example.com"))
    .thenAccept(System.out::println);
```

## Architecture / How It Works

A policy is a small configuration object; the actual work happens inside a **`FailsafeExecutor`**, produced by `Failsafe.with(...)`. Each policy contributes a `PolicyExecutor` stage, and the stages are assembled into a chain that wraps your `Supplier`/`Runnable`. When you call `.get()`, execution passes down the chain to your logic and results propagate back up, each stage deciding whether to return, retry, short-circuit, or convert the outcome.

**Composition order is significant and left-to-right = outer-to-inner.** In `Failsafe.with(fallback, retryPolicy, circuitBreaker)`, the fallback is outermost and the circuit breaker innermost, so the breaker records each individual attempt and the retry policy re-runs *through* the breaker. Reversing the order changes the semantics entirely — e.g. whether a retry sees the breaker's open state per attempt or only once. This ordering is the single most common source of misconfiguration.

**Some policies are stateful and must be shared across executions; others are not.** `RetryPolicy`, `Timeout`, and `Fallback` are effectively stateless descriptions and can be rebuilt freely. `CircuitBreaker`, `RateLimiter`, and `Bulkhead` hold live state (breaker open/closed status, token buckets, permit counts) and are meaningful only when a single instance is reused across all calls it is meant to govern. Constructing a fresh `CircuitBreaker` per request is a real and common bug: the breaker can never accumulate enough failures to trip.

**Async execution** runs on `ForkJoinPool.commonPool()` unless you supply an executor via `.with(executor)`. `getAsyncExecution` exposes manual control for callbacks that must signal completion themselves (e.g. wrapping a callback-based client). Event listeners (`onRetry`, `onFailure`, `onSuccess`, `onCircuitOpen`, etc.) fire at each policy boundary and are the integration point for logging and metrics — there is no built-in metrics backend. Optional modules `failsafe-okhttp` and `failsafe-retrofit` adapt the executor to those HTTP clients.

## Production Notes

- **Reuse stateful policy instances.** Store `CircuitBreaker`/`RateLimiter`/`Bulkhead` as singletons (or per logical resource). A new instance per call is the classic footgun.
- **Get composition order right, and test it.** Write a test that forces the breaker open and asserts retries behave as intended; ordering bugs do not surface on the happy path.
- **Async work occupies the common pool.** CPU-bound or blocking work run through `getAsync` without a dedicated executor can starve the JVM-wide `ForkJoinPool.commonPool()`. Pass an explicit, sized executor for anything non-trivial.
- **Timeouts and interruption.** A `Timeout` with `withInterrupt(true)` interrupts the executing thread; without it, a timed-out task keeps running to completion in the background and only its *result* is discarded. Blocking I/O that ignores interrupts will not actually stop.
- **Retries amplify load.** A retry policy in front of a struggling downstream can turn a brownout into an outage. Pair retries with a circuit breaker and jittered backoff (`withJitter`) rather than shipping retry alone.
- **The v2 → v3 migration is not a drop-in bump.** Both the Maven coordinates (`net.jodah` → `dev.failsafe`) and the Java package names changed, and builder-style construction replaced several constructors. Expect a mechanical but non-zero code change[^2].
- **Release cadence has slowed.** The last tagged release, 3.3.2, is from June 2023[^3]; the `master` branch still receives occasional commits (last push December 2025) but no new versioned release has followed. Treat it as stable-and-quiet rather than rapidly evolving, and pin your version.

## When to Use / When Not

**Use when:**
- You want resilience policies with zero added dependencies and a small footprint.
- You need programmatic, framework-agnostic control over retry/breaker/timeout logic.
- You want the same policy objects to serve both sync and `CompletableFuture` code paths.
- You are outside Spring, or want to avoid pulling in a metrics/reactive stack.

**Avoid when:**
- You want first-class Spring Boot starters, annotation-driven config, and Micrometer metrics out of the box — Resilience4j fits that better.
- You need a reactive-native operator model for Reactor/RxJava pipelines.
- You require an actively versioned dependency with frequent releases and a large contributor pool; Failsafe is stable but low-velocity.

## Alternatives

- resilience4j/resilience4j — functional resilience library for Java 8+; use it when you want modular artifacts, Spring Boot auto-config, and built-in Micrometer metrics.
- Netflix/Hystrix — the historical incumbent; in maintenance mode and not recommended for new projects.
- spring-projects/spring-retry — use it when you are already in Spring and only need annotation-based retry/circuit-breaker without the other policies.
- reactor/reactor-core — its `retryWhen`/`timeout` operators cover resilience natively if your code is already a `Mono`/`Flux` pipeline.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2016-11 | First stable release under `net.jodah:failsafe`[^3]. |
| 2.0.0 | 2019-01 | Major API rework; async improvements[^3]. |
| 3.0.0 | 2021-11 | Coordinates/package moved to `dev.failsafe`; builder API; RateLimiter, Bulkhead, and okhttp/retrofit modules[^2]. |
| 3.3.0 | 2022-09 | Later 3.x refinements[^3]. |
| 3.3.2 | 2023-06 | Latest tagged release as of this writing[^3]. |

## References

[^1]: Failsafe README and project authorship — "Copyright Jonathan Halterman and friends." https://github.com/failsafe-lib/failsafe
[^2]: Failsafe 3.0 migration — coordinate change from `net.jodah:failsafe` to `dev.failsafe:failsafe` and package rename. https://failsafe.dev/faqs/
[^3]: Failsafe git tags / release history (dates from tag commits). https://github.com/failsafe-lib/failsafe/tags

## Tags

java, jvm, resilience, fault-tolerance, circuit-breaker, retry, rate-limiter, bulkhead, timeout, fallback, zero-dependency
