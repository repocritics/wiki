# ReactiveX/RxJava

> Reactive Extensions for the JVM — composing asynchronous, event-based programs as observable sequences of data.

[GitHub repo](https://github.com/ReactiveX/RxJava) ·
[Official website](http://reactivex.io) ·
[License: Apache-2.0](https://github.com/ReactiveX/RxJava/blob/3.x/LICENSE)

## Overview

RxJava is the JVM port of the ReactiveX API — the observer pattern extended to
sequences of data/events, plus a large operator vocabulary for composing those
sequences declaratively while abstracting threading, synchronization, and
backpressure[^1]. It began as a Netflix project (Ben Christensen, Jafar Husain)
open-sourced in 2013, and has been driven for most of its life by a single
lead maintainer, David Karnók (`akarnokd`)[^2]. At ~48k stars it is one of the
most-depended-on libraries in the JVM ecosystem.

The defining design decision — introduced in the 2.x rewrite — is the split
between `Observable` (no backpressure, for short or GUI-bound streams) and
`Flowable` (Reactive Streams-compliant, backpressure-aware, for potentially
unbounded async sources). Choosing wrong is the single most common source of
`MissingBackpressureException` in production. The library is null-hostile:
emitting `null` throws, a deliberate break from 1.x.

RxJava's honest tension in 2026 is positional, not technical. It is mature,
correct, and still actively maintained — 4.0 is in active development targeting
Java 26 and `java.util.concurrent.Flow`, with virtual-thread schedulers and a
new blocking `Streamable` type[^3]. But on Android its mindshare has been
largely absorbed by Kotlin Coroutines/Flow, and on the server by Project Reactor
inside the Spring stack. New greenfield JVM projects rarely reach for it; the
large existing install base is why it remains maintained.

## Getting Started

Add the dependency (3.x is the current stable line; 4.x is pre-release):

```groovy
implementation "io.reactivex.rxjava3:rxjava:3.1.11"
```

A minimal flow moving work to a background thread and results to another:

```java
import io.reactivex.rxjava3.core.Flowable;
import io.reactivex.rxjava3.schedulers.Schedulers;

Flowable.fromCallable(() -> {
        Thread.sleep(1000);          // pretend expensive work
        return "Done";
    })
    .subscribeOn(Schedulers.io())    // run the source on an I/O thread
    .observeOn(Schedulers.single())  // deliver results on another thread
    .subscribe(System.out::println, Throwable::printStackTrace);

Thread.sleep(2000); // default schedulers run on daemon threads
```

Package coordinates changed across major versions: `rx.*` (1.x), `io.reactivex.*`
(2.x), `io.reactivex.rxjava3.*` (3.x), `io.reactivex.rxjava4.*` (4.x).

## Architecture / How It Works

Five base reactive types, discoverable by cardinality and protocol:

- `Flowable<T>` — 0..N with backpressure, implements Reactive Streams `Publisher`.
- `Observable<T>` — 0..N without backpressure.
- `Single<T>` — exactly one item or an error.
- `Maybe<T>` — zero or one item, or an error.
- `Completable` — completion or error only, no items.

A flow has distinct lifecycle phases: **assembly time** (operators are chained,
nothing runs, no side effects), **subscription time** (`subscribe()` wires the
chain and fires `doOnSubscribe`), and **runtime** (items, errors, and terminal
signals flow). Confusing assembly-time evaluation with runtime is a classic bug —
e.g. `Single.just(counter.get())` reads the counter at assembly, before the
upstream ran; `Single.defer(...)` or `fromCallable(...)` fixes it.

Concurrency is expressed through `Scheduler`s rather than raw `Thread`s or
`ExecutorService`s. `subscribeOn` chooses where the source runs (it moves the
whole upstream, and only the first one in a chain wins); `observeOn` switches the
thread for everything downstream of it, and can appear multiple times. Standard
schedulers: `computation()` (fixed pool sized to CPUs), `io()` (unbounded,
elastic — can exhaust threads), `single()`, `trampoline()`, `newThread()`.

Operators return new immutable instances (fluent builder shape); a `Flowable`
is never mutated in place. Parallelism is not implicit — a chain is sequential
by default, and true parallel work is expressed by fanning out with `flatMap`
into inner flows (unordered) or the `parallel()`/`ParallelFlowable` API.

## Production Notes

**Disposal and leaks.** Subscriptions are live resources. On Android, a
subscription that captures an `Activity`/`View` and outlives it leaks the whole
view tree. The standard defense is a `CompositeDisposable` cleared in the
lifecycle teardown (`onDestroy`/`onCleared`). This is the most frequent RxJava
production bug, and the reason `autodispose`/`RxLifecycle` exist.

**Observable vs Flowable.** Use `Observable` for bounded or user-driven streams
(clicks, small lists). Use `Flowable` when a source can outrun its consumer.
Getting this wrong surfaces as `MissingBackpressureException` under load, often
only in production traffic, not tests.

**Undeliverable exceptions.** After a chain is disposed, an error with nowhere
to go is routed to the global `RxJavaPlugins.onError` handler and, by default,
rethrown on the emitting thread — which can crash the process. Long-running
apps should install a global error handler that logs and swallows benign
post-dispose errors while still surfacing genuine bugs.

**Opaque stack traces.** Errors deep in a long operator chain produce stack
traces dominated by internal frames with little assembly context. Debugging aids
(`.doOnError`, assembly tracking via `RxJavaPlugins`) exist but the ergonomics
are worse than imperative code or coroutine stack traces.

**No nulls.** Since 2.x, emitting `null` throws `NullPointerException`. Code
ported from 1.x that relied on nullable emissions needs `Optional`-style wrapping
or a sentinel.

**Migration cost.** 1.x → 2.x was a full rewrite (new package, new types,
Reactive Streams alignment) and could not be done incrementally within a class;
the `rxjava2-interop` bridge exists for mixed graphs. 2.x → 3.x was lighter but
still a package rename (`io.reactivex` → `io.reactivex.rxjava3`) that touches
every import. Plan major upgrades as their own project.

## When to Use / When Not

**Use when:**
- You have an existing RxJava/RxAndroid codebase — stay consistent, it is maintained.
- You need a mature, cross-JVM (not Kotlin-only) operator set for stream composition.
- You need Reactive Streams interop with a non-Spring reactive library.
- You want fine-grained backpressure control over unbounded async sources.

**Avoid when:**
- You are on Kotlin/Android greenfield — Coroutines + Flow are the idiomatic choice.
- You are inside Spring/WebFlux — Project Reactor is the native, integrated stack.
- Your async is simple (a few callbacks) — the operator learning curve is not worth it.
- Your team lacks Rx experience and the codebase is small; the footguns dominate.

## Alternatives

- Kotlin/kotlinx.coroutines (Flow) — use instead for Kotlin/Android; language-level
  suspension, readable stack traces, structured concurrency.
- reactor/reactor-core — use instead inside Spring/WebFlux; `Mono`/`Flux` are the
  first-class types there.
- ReactiveX/RxKotlin — thin Kotlin idioms over RxJava; a companion, not a replacement.
- ReactiveX/RxAndroid — Android schedulers for RxJava; companion, not a substitute.
- smallrye/smallrye-mutiny — use instead in Quarkus/reactive-messaging stacks.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013-01 | Repo opened; Netflix open-sources the JVM ReactiveX port[^2]. |
| 1.0 | 2014-11 | First stable line, `rx.*` package. |
| 2.0 | 2016-11 | Full rewrite: Reactive Streams, `Flowable` vs `Observable`, null-hostile[^4]. |
| 3.0 | 2020-02 | Java 8 baseline, package `io.reactivex.rxjava3`, cleanup of 2.x[^5]. |
| 4.0 | 2026-11 (target) | Java 26, `j.u.c.Flow`-based, virtual-thread schedulers, `Streamable`[^3]. |

## References

[^1]: RxJava README — Reactive Extensions for the JVM. https://github.com/ReactiveX/RxJava
[^2]: ReactiveX project & RxJava history. http://reactivex.io/intro.html
[^3]: RxJava 4.0.0 milestone and README roadmap (Java 26, virtual threads, `Streamable`). https://github.com/ReactiveX/RxJava/milestone/30
[^4]: RxJava 2.0 "What's different in 2.0" wiki. https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0
[^5]: RxJava 3.0 release notes. https://github.com/ReactiveX/RxJava/releases/tag/v3.0.0

## Tags

java, jvm, reactive, reactive-streams, reactivex, backpressure, asynchronous, observable, concurrency, android, event-driven
