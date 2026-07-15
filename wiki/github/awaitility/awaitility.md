# awaitility/awaitility

> A small Java DSL for testing asynchronous systems by polling a condition until it becomes true or a timeout expires.

[GitHub repo](https://github.com/awaitility/awaitility) ·
[Documentation wiki](https://github.com/awaitility/awaitility/wiki) ·
[License: Apache-2.0](https://github.com/awaitility/awaitility/blob/master/LICENSE)

## Overview

Awaitility is a test-support library that solves one narrow problem: asserting on state that becomes true *eventually* rather than immediately. When a test publishes a message, kicks off a background thread, or triggers an event handler, the result is not available on the next line. The naive fixes — `Thread.sleep(5000)` or hand-rolled retry loops — are either slow, flaky, or both. Awaitility replaces them with a declarative DSL: `await().atMost(5, SECONDS).until(() -> repository.count() == 1)`[^1]. It was created by Johan Haleby and has been maintained on the same core API for over a decade (first commit 2010).

The mechanism is **polling, not notification**. Awaitility does not hook into your async framework, register callbacks, or observe futures. It repeatedly evaluates a supplied condition (a `Callable<Boolean>`, a Hamcrest matcher, or an assertion block) on a fixed interval until it passes or the deadline is hit. This is the library's defining tradeoff: polling is universal — it works against any async substrate (JMS, Kafka, threads, files, HTTP, database rows) without integration — but it is also inherently lossy and adds latency. A condition that is true for only 10 ms between two 100 ms polls will never be observed, and a condition that becomes true 1 ms after a poll costs you the full remaining interval.

Awaitility is a *test-scope* tool. It is not a production retry/backoff library and should not be used to drive application control flow; libraries like failsafe or resilience4j exist for that. Its audience is JVM developers writing integration and end-to-end tests where the system under test has genuine asynchronicity.

## Getting Started

Maven:

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.3.1</version>
    <scope>test</scope>
</dependency>
```

Gradle:

```groovy
testImplementation 'org.awaitility:awaitility:4.3.1'
```

```java
import static org.awaitility.Awaitility.await;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.hamcrest.Matchers.equalTo;

@Test
void customerStatusIsUpdatedAsynchronously() {
    messageBroker.publishMessage(updateStatusMessage);

    // Poll until the lambda returns true, or fail after 5 seconds.
    await().atMost(5, SECONDS)
           .until(() -> customerRepository.statusOf(id) == Status.UPDATED);

    // Or poll a supplier against a Hamcrest matcher:
    await().atMost(5, SECONDS)
           .until(customerRepository::count, equalTo(1L));
}
```

For re-running a full assertion block (e.g. AssertJ/JUnit assertions) until it stops throwing, use `untilAsserted`:

```java
await().atMost(5, SECONDS).untilAsserted(() ->
    assertThat(customerRepository.findById(id)).isPresent());
```

## Architecture / How It Works

The core is a **condition-evaluation loop** driven by a `ConditionFactory` (what `await()` returns). Configuration methods (`atMost`, `pollInterval`, `pollDelay`, `during`, `ignoreExceptions`, `conditionEvaluationListener`) are chained fluently and produce an immutable, reconfigured factory; the terminal `until*` call starts the loop.

Default timings, unless overridden: **poll interval 100 ms, initial poll delay equal to the poll interval (100 ms), timeout 10 seconds**[^2]. The first evaluation therefore happens *after* the poll delay, not at t=0 — a subtle behavior when the condition is already true on entry. `pollDelay(Duration.ZERO)` forces an immediate first check.

Evaluation runs on a separate poll thread by default (via an internal executor), which is what lets Awaitility enforce a hard timeout even if a single condition evaluation hangs. `pollInSameThread()` disables this — necessary when the condition touches thread-confined state (e.g. Spring's `@Transactional` test context, or thread-local security contexts) that does not exist on the poll thread.

Three condition shapes:
- **Boolean conditions** (`until(Callable<Boolean>)`) — poll until `true`.
- **Matcher conditions** (`until(Supplier, Matcher)`) — poll a value supplier until it satisfies a Hamcrest matcher; failure messages report the last polled value.
- **Assertion conditions** (`untilAsserted(ThrowingRunnable)`) — run a block that throws (AssertionError) until it completes without throwing. Exceptions are swallowed during the wait and only the final one surfaces on timeout.

By default an evaluation that throws an exception fails the wait immediately; `ignoreExceptions()` / `ignoreExceptionsInstanceOf(...)` treat exceptions as "not yet true" and keep polling (this is the default behavior *within* `untilAsserted`). Supplementary predicates: `during(...)` (condition must hold for a sustained period), `atLeast(...)`/`between(...)` (fail if the condition passes too early), and a Fibonacci/custom poll-interval strategy for backoff. Companion artifacts `awaitility-kotlin`, `awaitility-scala`, and `awaitility-groovy` add idiomatic wrappers.

## Production Notes

**It is a polling loop — cost and blind spots are structural.** A test that waits on a condition true 1 ms after a poll still pays up to a full poll interval of latency; across a large suite this adds real wall-clock time. Conversely, transient states shorter than the poll interval are invisible. Tune `pollInterval` down for tight timing tests, but never below what the system can actually produce, or you burn CPU polling.

**`untilAsserted` re-runs its block every poll.** The assertion body must be idempotent and side-effect-free. A block that consumes from a queue, increments a counter, or mutates the system under test will behave differently on each poll and produce confusing failures. Read-only assertions only.

**Thread confinement is the most common footgun.** Because polling defaults to a separate thread, conditions that depend on thread-local state (transactions, `SecurityContextHolder`, JUnit/Spring test-scoped resources) will see the wrong or missing context and fail nondeterministically. Reach for `pollInSameThread()` in those cases; be aware it removes the hard-timeout guarantee for a hung evaluation.

**Timeout vs. poll delay confusion.** Because the default poll delay is 100 ms, `await().until(alreadyTrue)` still waits ~100 ms. Use `pollDelay(Duration.ZERO)` when the condition may already hold. Setting `atMost` shorter than `pollDelay` is a misconfiguration that fails immediately.

**Version/JDK boundaries.** The 4.x line targets Java 8+ and uses `java.time.Duration` throughout; the older 2.x/3.x lines predate that and used a custom `Duration` type, so upgrading across the 3→4 boundary means rewriting duration literals and static imports[^3]. Global defaults (`Awaitility.setDefaultTimeout`, etc.) are process-wide mutable statics — convenient in a base test class, but they leak across tests if not reset, and can surprise parallel test runners. As of 4.3.1 (2026-04) these defaults can also be set via system properties[^4].

**CI flakiness.** Fixed timeouts that pass on a fast laptop fail on a loaded CI runner. Prefer generous `atMost` values with a short `pollInterval` (fast when the system is fast, patient when it is slow) over a tight timeout tuned to local hardware.

## When to Use / When Not

**Use when:**
- You are integration-testing genuinely asynchronous behavior (message queues, event handlers, background jobs, eventual-consistency reads) on the JVM.
- You want to delete `Thread.sleep` calls and flaky manual retry loops from a test suite.
- The async substrate has no clean synchronization hook you can await directly.

**Avoid when:**
- The operation exposes a `Future`, `CompletableFuture`, or reactive publisher you can deterministically await/verify — do that instead of polling; it is exact and faster (see reactor-test's `StepVerifier` for reactive code).
- You need production retry/backoff/circuit-breaking — this is a test library, not a resilience layer.
- The behavior is actually synchronous; polling just adds latency and hides the fact that no wait is needed.
- You need to observe a transient intermediate state reliably — polling can miss it.

## Alternatives

- reactor/reactor-core (reactor-test) — for Project Reactor / reactive streams, `StepVerifier` verifies emissions deterministically without polling. Use it when the code under test is a `Flux`/`Mono`.
- failsafe-lib/failsafe — production retry, timeout, and circuit-breaker policies. Use it when you need resilience in application code, not test assertions.
- resilience4j/resilience4j — production fault-tolerance (retry, rate limiter, bulkhead). Use instead of awaitility whenever the loop belongs in the app, not the test.
- kotest/kotest — Kotlin test framework whose `eventually { }` block covers awaitility's niche natively. Use it when you are already on kotest in a Kotlin codebase.
- JUnit 5 `assertTimeout` — bounds how long a synchronous call may take. Use it to fail slow code, not to wait for eventual state (it does not poll).

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2010–2011 | Initial DSL, Hamcrest-based conditions. |
| 2.0 | 2016 | API cleanup; field/proxy condition helpers. |
| 3.0 | 2017 | Java 8 baseline, lambda-friendly conditions. |
| 4.0 | 2019 | Switched public API to `java.time.Duration`[^3]. |
| 4.2.0 | 2023 | Maintenance, dependency and JDK-compatibility updates. |
| 4.2.2 | 2024-08-07 | Support for "ea" (early-access) JVM versions[^4]. |
| 4.3.0 | 2025-02-21 | Improved Kotlin time support; new `untilAsserted` path[^4]. |
| 4.3.1 | 2026-04-17 | Configurable default timeout/poll via system properties[^4]. |

## References

[^1]: Awaitility README and Usage Guide. https://github.com/awaitility/awaitility/wiki/Usage
[^2]: Awaitility defaults (poll interval, poll delay, timeout). https://github.com/awaitility/awaitility/wiki/Usage#defaults
[^3]: Awaitility changelog (2.x/3.x/4.0 migration to `java.time.Duration`). https://github.com/awaitility/awaitility/raw/master/changelog.txt
[^4]: Awaitility News/changelog, 4.2.2–4.3.1 releases. https://github.com/awaitility/awaitility/blob/master/changelog.txt

## Tags

java, testing, asynchronous, integration-testing, polling, dsl, jvm, test-tooling, concurrency, eventual-consistency
