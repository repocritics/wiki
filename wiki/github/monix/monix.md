# monix/monix

> ReactiveX-style asynchronous streams and effect types for Scala and Scala.js —
> a co-parent of Cats-Effect whose stable line remains on the pre-2021 effect stack.

[GitHub repo](https://github.com/monix/monix) ·
[Official website](https://monix.io) ·
[License: Apache-2.0](https://github.com/monix/monix/blob/main/LICENSE.txt)

## Overview

Monix is a Scala / Scala.js library for composing asynchronous, event-based
programs. It began in 2014 as Monifu, a back-pressured implementation of the
ReactiveX model for Scala, and grew into a full effect system: `Task` (lazy
async effect), `Coeval` (lazy sync), `Observable` (push-based streams),
`Iterant` (pull-based streams), plus low-level primitives like `Scheduler`,
`Cancelable`, and `Local`[^1]. It is a Typelevel project and — via its author
Alexandru Nedelcu — one of the design parents and original implementors of
Cats-Effect[^2].

That history is also the defining tension. Monix's `Task` shaped what became
`cats.effect.IO`, but when the Typelevel ecosystem moved to Cats-Effect 3 in
2021[^3], Monix's stable 3.x series stayed compatible with Cats-Effect 2.x
only; a CE3-compatible Monix 4 was worked on but never reached a stable
release. The result: `Task` and `Iterant` have been largely superseded by
`cats.effect.IO` and fs2 in new Typelevel codebases, while `Observable` —
a genuinely distinct push-based, back-pressured ReactiveX surface with no
direct equivalent in fs2 — remains the main reason the library is still used.

Activity reflects maintenance mode rather than abandonment: ~1.9k stars, and
the last push (June 2026) is dependency and Scala-version upkeep (Scala 2.13 /
3.3 LTS, JDK 17 baseline) rather than new features. 86 open issues sit against
a small maintainer bench.

## Getting Started

```scala
// build.sbt — stable 3.x line (Cats-Effect 2.x compatible)
libraryDependencies += "io.monix" %% "monix" % "3.4.1"
```

```scala
import monix.eval.Task
import monix.reactive.Observable
import monix.execution.Scheduler.Implicits.global
import scala.concurrent.duration._

object Main extends App {
  val stream: Task[Long] =
    Observable
      .interval(100.millis)      // cold stream, one tick per subscriber
      .take(10)
      .mapEval(i => Task(i * 2)) // sequence an effect per element
      .foldLeftL(0L)(_ + _)      // materialize to Task

  // Nothing runs until a Scheduler executes the Task:
  println(stream.runSyncUnsafe()) // 90
}
```

Monix is modular: `monix-execution` (Scheduler, atomics), `monix-catnap`
(Cats-Effect-typed concurrency: `CircuitBreaker`, `MVar`, `ConcurrentQueue`),
`monix-eval` (`Task`, `Coeval`), `monix-reactive` (`Observable`), `monix-tail`
(`Iterant`), or the `monix` umbrella for everything[^1].

## Architecture / How It Works

**Task** is a trampolined run-loop interpreter over a description of a
computation — nothing executes at construction. Execution requires a
`Scheduler`, Monix's enriched `ExecutionContext` that adds scheduled/delayed
execution, error reporting, and an execution model setting. That last one is a
real behavioral knob: by default Monix batches synchronous steps and inserts
async boundaries periodically to preserve fairness, and you can force
fully-synchronous or always-async models per Task with `executeWithModel`.
Cancellation is first-class — every `runToFuture` yields a `CancelableFuture`,
and `Task` race/timeout combinators propagate cancellation.

**Observable** is the ReactiveX side. Its back-pressure protocol is the
distinctive internal: a subscriber's `onNext` returns `Future[Ack]`
(`Continue` or `Stop`), so producers await downstream demand without a
separate request-counting channel like Reactive Streams' `Subscription` —
though converters to and from the Reactive Streams protocol are provided[^4].
Observables are cold by default; hot multicasting goes through `Subject`
variants (publish, behavior, replay) with the usual ReactiveX semantics.

**Iterant** (`monix-tail`) is the opposite pole: a purely functional,
pull-based stream parameterized over an arbitrary effect `F[_]` via
Cats-Effect type classes — conceptually the same territory as fs2's `Stream`.

The coupling story: `monix-execution` is dependency-light and usable alone;
`monix-catnap`, `monix-eval`, and `monix-tail` bind to Cats and Cats-Effect
**2.x** type classes. That pin is transitive — pulling Monix 3.x into a
project sets your Cats-Effect major version.

## Production Notes

- **The CE2 pin is the dominant constraint.** http4s, doobie, fs2, and the
  rest of the Typelevel stack have required Cats-Effect 3 since 2021[^3].
  Mixing Monix `Task` with current versions of those libraries is not
  possible in one dependency graph; teams either freeze the old stack or
  confine Monix to `Observable` + `monix-execution` and run CE3 `IO` beside
  it with manual interop.
- **Maintenance cadence.** Commits in 2024–2026 are Scala/JDK/dependency
  bumps. Expect bug-fix triage to be slow; do not expect new features. Budget
  for an eventual migration (fs2, Pekko Streams, or ZIO streams) in any new
  system design.
- **`Local` propagation is opt-in.** `TaskLocal` / `Local` (used for MDC-style
  context and tracing) does not propagate across async boundaries unless the
  task runs with `Task.defaultOptions.enableLocalContextPropagation` — a
  classic source of "my correlation ID disappeared" bugs.
- **Scheduler discipline.** The default global Scheduler is fine for CPU-bound
  work; blocking calls need a dedicated `Scheduler.io()` or wrapping — same
  rules as any effect system, but Monix's auto-batching execution model can
  mask starvation in tests and reveal it under production load.
- **Scala.js caveats.** `runSyncUnsafe` and anything blocking are unavailable
  on JS; code meant to cross-compile must stay in `runToFuture` /
  callback territory.
- **Hot-stream leaks.** Replay/behavior subjects retain buffered elements for
  late subscribers; unbounded replay on a long-lived subject is a slow memory
  leak, as in every ReactiveX implementation.

## When to Use / When Not

**Use when:**
- You maintain an existing Monix codebase — the 3.4.x line is stable and still
  receives compatibility upkeep.
- You specifically want ReactiveX push semantics (hot streams, subjects, a
  large operator vocabulary) with back-pressure in Scala.
- You need `monix-execution` alone: `Scheduler`, `Atomic`, and
  `CancelableFuture` are useful without buying the effect stack.

**Avoid when:**
- You are starting a new Typelevel-stack service — `cats.effect.IO` + fs2 is
  where the ecosystem, docs, and hiring pool are.
- You need Cats-Effect 3 compatibility in the same dependency graph.
- You want typed errors in the effect (`IO[E, A]`): that lives in the separate
  monix/monix-bio project or in ZIO, not here[^5].
- You expect active feature development or fast issue turnaround.

## Alternatives

- typelevel/cats-effect — use instead for any new purely-functional Scala
  service; CE3 `IO` is the direct successor to `Task` in ecosystem terms.
- typelevel/fs2 — use instead of `Iterant`, and instead of `Observable` when
  pull-based streaming fits; the standard CE3 streaming library.
- zio/zio — use instead when you want typed errors, built-in streams, and an
  integrated ecosystem outside Typelevel.
- apache/pekko — use instead for Reactive-Streams-based stream graphs in the
  open-source continuation of Akka after Akka's 2022 BSL relicense[^6].
- ReactiveX/RxJava — use instead on the JVM without Scala when you want the
  same ReactiveX model Monix implements.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x (Monifu) | 2014 | Project started as Monifu: ReactiveX-inspired, back-pressured streams for Scala/Scala.js. |
| 1.0 | 2015 | First stable release; project renamed Monifu → Monix around this point. |
| 2.0 | 2016 | `Task` and `Coeval` introduced; codebase split into sub-modules. |
| 3.0 | 2019-09 | Cats-Effect (2.x) integration; `Iterant` / monix-tail; monix-catnap. |
| 3.4 | 2021 | Scala 3 support added to the stable line. |
| 4.x | — | CE3-compatible series developed but never reached a stable release; 3.4.x remains the maintained line. |

## References

[^1]: Monix documentation — current usage and sub-modules graph. https://monix.io/docs/current/
[^2]: monix/monix README — "one of the parents and implementors of Cats Effect". https://github.com/monix/monix#overview
[^3]: Typelevel, Cats-Effect 3 release (2021) and migration guide. https://typelevel.org/cats-effect/docs/migration-guide
[^4]: Monix Observable documentation (back-pressure protocol, Reactive Streams interop). https://monix.io/docs/current/reactive/observable.html
[^5]: Monix BIO — `IO[E, A]` with typed errors, separate project. https://bio.monix.io/docs/introduction
[^6]: Lightbend, "Why We Are Changing the License for Akka" — 2022-09-07. https://www.lightbend.com/blog/why-we-are-changing-the-license-for-akka

## Tags

scala, scala-js, reactive-streams, reactive-programming, functional-programming, async, concurrency, streaming, cats-effect, typelevel, observable
