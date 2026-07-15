# resilience4j/resilience4j

> A modular fault-tolerance library for the JVM: circuit breaker, rate limiter, bulkhead, retry, and time limiter as composable function decorators.

[GitHub repo](https://github.com/resilience4j/resilience4j) ·
[User Guide](https://resilience4j.readme.io/docs) ·
[License: Apache-2.0](http://www.apache.org/licenses/LICENSE-2.0.txt)

## Overview

Resilience4j is a Java fault-tolerance library that emerged as the de facto successor to Netflix Hystrix after Hystrix entered maintenance mode in 2018[^1]. Where Hystrix wrapped protected calls in command objects and relied heavily on a dedicated thread pool per dependency, Resilience4j takes a functional approach: each resilience pattern is a higher-order decorator that wraps any `Supplier`, `Function`, `Runnable`, or method reference, and decorators can be stacked in whatever combination a call needs.

The defining design choice is modularity. The core patterns ship as separate artifacts (`resilience4j-circuitbreaker`, `-ratelimiter`, `-bulkhead`, `-retry`, `-timelimiter`, `-cache`), so an application pulls in only what it uses and carries no framework baggage for the rest. This keeps the dependency footprint small but pushes composition responsibility onto the caller — the order in which you stack decorators is semantically significant and is the single most common source of surprise (see Production Notes).

The library targets teams protecting calls to remote services and other failure-prone boundaries. It is heavily used through its Spring Boot starters, where the same patterns are applied declaratively via annotations and YAML rather than assembled by hand. The tradeoff versus a heavier framework like Hystrix is fewer opinions and less magic in exchange for more explicit wiring.

## Getting Started

```groovy
// Gradle — pick individual modules, or resilience4j-all for everything
implementation "io.github.resilience4j:resilience4j-circuitbreaker:2.2.0"
implementation "io.github.resilience4j:resilience4j-retry:2.2.0"
```

```java
// Stack a CircuitBreaker and Retry around a call, with a fallback.
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendService");
Retry retry = Retry.ofDefaults("backendService");

Supplier<String> decorated = Decorators.ofSupplier(() -> backendService.doSomething())
    .withCircuitBreaker(circuitBreaker)
    .withRetry(retry)
    .decorate();

String result;
try {
    result = decorated.get();
} catch (Exception e) {
    result = "fallback";   // CallNotPermittedException when the circuit is OPEN
}
```

For async work, `Decorators.ofSupplier` also composes with a `ThreadPoolBulkhead` and a `TimeLimiter` over a `CompletableFuture`, since `TimeLimiter` can only enforce a deadline on an async result.

## Architecture / How It Works

Each pattern is a small state machine plus a decorator that consults it:

- **CircuitBreaker** tracks call outcomes in a lock-free sliding window — either count-based (last N calls) or time-based (last N seconds of aggregated buckets). It transitions between `CLOSED`, `OPEN`, and `HALF_OPEN` on failure-rate and slow-call-rate thresholds, with additional `DISABLED`, `FORCED_OPEN`, and `METRICS_ONLY` states. State is held in atomic references, so the hot path takes no locks.
- **RateLimiter** defaults to `AtomicRateLimiter`, a lock-free token-refresh algorithm dividing time into cycles; a `SemaphoreBasedRateLimiter` is also available.
- **Bulkhead** comes in two flavors: `SemaphoreBulkhead` (default) caps concurrent executions on the caller's own thread, while `ThreadPoolBulkhead` runs work on a bounded pool with its own queue, giving true thread isolation like Hystrix.
- **Retry** and **TimeLimiter** are stateless per-call helpers; retries add configurable wait/backoff, time limits wrap a `CompletableFuture`.

Instances are created and reused through **Registry** objects (`CircuitBreakerRegistry`, etc.), which hold a shared default config and mint named instances on demand. This is how the Spring Boot integration lazily materializes an instance the first time a configured name is referenced. Every instance emits an event stream (state transitions, successes, failures, retries) that add-on modules bridge to Micrometer, Dropwizard Metrics, or Prometheus.

Historically the library depended on Vavr (formerly Javaslang) for its functional types, and older examples use `Try.ofSupplier(...).recover(...)`. Version 2.0 removed the Vavr dependency from the core to cut transitive weight[^2], which is why current code typically uses plain try/catch or `Decorators` fallbacks instead.

## Production Notes

**Decorator order is not commutative.** `Retry(CircuitBreaker(call))` retries only calls the breaker permitted and counts each failed attempt toward the breaker; `CircuitBreaker(Retry(call))` records one outcome after retries are exhausted. In Spring, the aspect order is fixed by default (Retry outermost, then CircuitBreaker, RateLimiter, TimeLimiter, Bulkhead innermost) and is configurable via `*.order` properties — getting this wrong silently changes failure semantics.

**ThreadPoolBulkhead breaks ThreadLocal propagation.** Because work executes on a pool thread, MDC logging context, `SecurityContext`, and transaction context do not carry over unless you propagate them explicitly. The SemaphoreBulkhead keeps the caller's thread and avoids this, but provides no timeout isolation.

**TimeLimiter does not interrupt synchronous blocking calls.** It enforces a deadline on a `CompletableFuture`; to actually abandon a slow blocking call you must run it on a `ThreadPoolBulkhead` (or another executor) so the future can be cancelled. Wrapping a plain synchronous `Supplier` in a TimeLimiter alone will not stop it.

**Instance sharing is a footgun.** A CircuitBreaker, Bulkhead, or RateLimiter shared across multiple backends means one backend's failures or load open the circuit / exhaust the limit for all of them. The maintainers' guidance is one uniquely named instance per protected dependency, even for stateless patterns like Retry, so metrics stay attributable[^3].

**Version and JVM floors move.** 1.x ran on Java 8; 2.x required Java 17 and dropped Vavr; 3.x requires Java 21 and adds an opt-in to route internal schedulers onto virtual threads[^4]. Upgrades across these lines are not drop-in — expect package/dependency changes, and confirm the Spring Boot starter version matches your Boot generation (`resilience4j-spring-boot2` vs `-spring-boot3`).

## When to Use / When Not

**Use when:**
- You need selective, composable fault tolerance for JVM calls to remote services and want a small dependency footprint.
- You are on Spring Boot and want circuit breaking / retry / rate limiting configured declaratively.
- You are migrating off Hystrix and want a maintained equivalent with similar (but optional) thread-pool isolation.

**Avoid when:**
- You are not on the JVM — use Polly (.NET), a Go equivalent, or a service-mesh policy instead.
- Your resilience concerns are better handled at the infrastructure layer (Istio/Envoy, an API gateway) across many services and languages.
- You want a single opinionated abstraction with no wiring decisions — Resilience4j deliberately hands composition and ordering to you.

## Alternatives

- Netflix/Hystrix — the historical predecessor; in maintenance mode, not recommended for new work.
- failsafe-lib/failsafe — lightweight JVM retry/circuit-breaker library; simpler surface, no Spring-first integration.
- alibaba/Sentinel — flow control and circuit breaking with a dashboard and rule server; heavier, strong in the Alibaba/Spring Cloud ecosystem.
- App-vNext/Polly — the .NET equivalent; use when your stack is .NET rather than the JVM.
- spring-projects/spring-retry — use when you only need retry/recover semantics inside Spring and nothing else.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2015-06 | Project started as a Hystrix-inspired functional library. |
| 1.0.0 | 2019-08 | First stable release; Java 8, Vavr-based functional API. |
| 2.0.0 | 2022-10 | Java 17 floor; Vavr dependency removed from core[^2]. |
| 3.0.0 | 2025 | Java 21 floor; opt-in virtual-thread schedulers[^4]. |

## References

[^1]: Netflix, "Hystrix: Latency and Fault Tolerance — now in maintenance mode." https://github.com/Netflix/Hystrix#hystrix-status
[^2]: Resilience4j releases — 2.0.0 removed the Vavr dependency. https://github.com/resilience4j/resilience4j/releases
[^3]: Resilience4j README, "Best Practices — Instance Management: When to Share vs. Not Share Instances." https://github.com/resilience4j/resilience4j
[^4]: Resilience4j README, "Virtual thread support (Java 21 Project Loom)." https://github.com/resilience4j/resilience4j
[^5]: Resilience4j User Guide. https://resilience4j.readme.io/docs

## Tags

java, jvm, fault-tolerance, circuit-breaker, rate-limiter, retry, bulkhead, resilience, spring-boot, microservices, hystrix-alternative
