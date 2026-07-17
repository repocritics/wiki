# dotnet/reactive

> Reactive Extensions (Rx) for .NET — LINQ over live, asynchronous event streams via `IObservable<T>`.

[GitHub repo](https://github.com/dotnet/reactive) ·
[Official website](http://reactivex.io) ·
[License: MIT](https://github.com/dotnet/reactive/blob/main/LICENSE)

## Overview

Rx.NET is the .NET implementation of the Reactive Extensions, a model for treating live event
streams as first-class, queryable sequences. Its core idea is the duality between pull-based
`IEnumerable<T>`/`IEnumerator<T>` and push-based `IObservable<T>`/`IObserver<T>`: the same LINQ
operators (`Where`, `Select`, `SelectMany`, `GroupBy`) apply, but items arrive on the source's
schedule rather than being pulled by the consumer[^1]. It originated inside Microsoft's Cloud
Programmability Group (Erik Meijer's team) and was open-sourced and later handed to the .NET
Foundation; the repo has been on GitHub since 2013 but the technology predates that by several years[^2].

The repository is actually a monorepo of four related libraries, all "LINQ over sequences of
things": **Rx.NET** (`System.Reactive`, the flagship), **AsyncRx.NET** (`System.Reactive.Async`,
an experimental `IAsyncObservable<T>` variant with real `async`/`await` observers),
**Ix.NET** (`System.Interactive`, extra LINQ operators for `IEnumerable<T>`), and
**System.Linq.Async** (the LINQ operator set for `IAsyncEnumerable<T>`)[^2]. Several ideas
pioneered here — `IAsyncEnumerable<T>`, `MinBy`/`MaxBy` — were later absorbed into the .NET base
class library itself.

The defining tension: Rx is conceptually elegant and battle-tested, but the `System.Reactive`
package carries a heavy historical burden around UI-framework coupling and packaging, and much of
the community's forward momentum has moved to a modern reimplementation (Cysharp/R3). Rx.NET is
maintained (currently by endjin's Ian Griffiths and Howard van Rooijen) but its evolution is
deliberately slow and compatibility-bound[^2].

## Getting Started

```bash
dotnet add package System.Reactive
```

```cs
using System;
using System.Reactive.Linq;

// A cold observable: one timer tick per second, projected and filtered with LINQ.
IObservable<long> ticks = Observable.Interval(TimeSpan.FromSeconds(1));

using IDisposable subscription = ticks
    .Where(n => n % 2 == 0)
    .Select(n => $"even tick {n}")
    .Subscribe(
        onNext: Console.WriteLine,
        onError: ex => Console.Error.WriteLine(ex),
        onCompleted: () => Console.WriteLine("done"));

// Disposing the subscription is how you unsubscribe — forgetting to is the classic Rx leak.
Console.ReadLine();
```

## Architecture / How It Works

Everything rests on two interfaces defined in the base class library, not in Rx itself:
`IObservable<T>` (a source you can `Subscribe` to) and `IObserver<T>` (the `OnNext`/`OnError`/
`OnCompleted` sink). Rx supplies the operators, the `Subject<T>` family (both source and sink),
and — critically — the **scheduler** abstraction (`IScheduler`) that decides *where and when*
work and notifications run. `ObserveOn`/`SubscribeOn` move a pipeline between the thread pool, a
UI dispatcher, or a virtual test scheduler[^1].

Two concepts trip up most newcomers:

- **Hot vs cold.** A cold observable (e.g. `Observable.Interval`, `Observable.Create`) starts
  work per subscriber, so two subscribers get independent runs. A hot observable (a `Subject`,
  or anything behind `Publish().RefCount()`) is already running and multicasts. Getting this
  wrong causes duplicated side effects or missed events.
- **Subscription is disposal.** `Subscribe` returns an `IDisposable`; the *only* way to stop is
  to dispose it. Long-lived subscriptions that outlive their intended scope are the standard
  Rx.NET memory leak, especially when a subscription closes over a UI control or a `Subject`
  never completes.

Rx has **no backpressure**. Unlike the Reactive Streams standard (Project Reactor, RxJava,
Akka Streams), an Rx observer cannot signal "slow down"; a fast producer feeding a slow consumer
either buffers (unbounded memory growth) or requires manual operators like `Sample`, `Throttle`,
`Buffer`, or `Window` to shed load. This is a deliberate design difference, not a bug, but it
means Rx.NET is a poor fit for flow-controlled data pipelines.

The `System.Reactive` package multi-targets a range of TFMs, and historically bundled
UI-framework schedulers (WPF `DispatcherScheduler`, WinForms) directly into the single package.
That coupling is the source of the packaging problems below.

## Production Notes

**The `System.Reactive` UI-dependency / packaging problem.** Because Rx 4.0 collapsed everything
into one `System.Reactive` package, and later versions added `-windows` TFMs carrying WPF/WinForms
scheduler types, referencing `System.Reactive` from an otherwise platform-neutral library could
drag in Windows Desktop framework references, inflate output, and cause friction in cross-platform,
trimmed, or AOT builds. This is a long-running, well-documented pain point; the v7.0 roadmap is
explicitly about splitting UI-specific pieces back out into separate packages so the core stays
lean[^3]. If you only need core Rx, be aware of what your TFM pulls in.

**Facade packages.** Old package names (`Rx-Main`, `Rx-Core`, `Rx-Linq`, `Rx-Interfaces`, and
similar) are empty tombstones redirecting to `System.Reactive`. New code should reference
`System.Reactive` directly; encountering the old names in a dependency graph signals stale
guidance.

**Subscription leaks and unhandled errors.** An `OnError` with no error handler is rethrown and
can crash the process on the scheduler thread. Always supply an error handler, and always own the
lifetime of every `IDisposable` returned by `Subscribe` (a `CompositeDisposable` or the
`.DisposeWith()` helpers in ReactiveUI are common patterns).

**Concurrency is explicit and easy to get wrong.** Rx does not add threads on its own; operators
run on whatever scheduler is in play. Debugging "why did this run on the wrong thread" usually
comes down to a missing or misplaced `ObserveOn`. For deterministic tests, use `TestScheduler`
(virtual time) rather than real timers.

**AsyncRx.NET is preview-only.** If you need genuinely asynchronous observers
(`IAsyncObservable<T>`), `System.Reactive.Async` exists but is explicitly experimental and not
recommended for production stability.

## When to Use / When Not

**Use when:**
- You have genuinely event-driven logic (UI input, live financial/telemetry feeds, sensor data)
  and want to compose it declaratively with LINQ operators.
- You need rich time-based operators — `Throttle`, `Debounce`/`Sample`, `Buffer`, `Window`,
  `CombineLatest` — that would be tedious to hand-roll.
- You are already in an Rx-based UI stack (ReactiveUI) or porting RxJS/RxJava concepts to .NET.

**Avoid when:**
- You just need to iterate an async sequence: `IAsyncEnumerable<T>` + `await foreach` (with
  `System.Linq.Async` for operators) is simpler and has natural backpressure.
- You need flow control / backpressure: use `System.Threading.Channels` instead.
- You are starting a new Unity/Godot or perf-sensitive project: evaluate Cysharp/R3 first, which
  was built to address Rx.NET's allocation and packaging issues.
- Your team is unfamiliar with hot/cold semantics and subscription lifetimes — the learning cliff
  is real and mistakes are silent.

## Alternatives

- Cysharp/R3 — modern reimplementation of Rx for .NET; use it when you want Rx semantics without
  the `System.Reactive` packaging baggage and with first-class Unity/Godot support.
- Cysharp/UniRx — Rx tailored to the Unity game loop; largely superseded by R3 but still in wide use.
- reactiveui/reactiveui — not a replacement but a complement: an MVVM framework built on top of Rx.
- dotnet/runtime (`System.Threading.Channels`) — use when you need bounded producer/consumer
  pipelines with backpressure rather than LINQ-over-events.
- ReactiveX/rxjs — the JavaScript sibling; reach for it when the reactive logic lives in the browser.

## History

| Version | Date | Notes |
|---------|------|-------|
| Rx 1.x | ~2011 | First stable release from Microsoft (originated in Erik Meijer's team)[^1]. |
| — | 2013 | Source moved to GitHub (this repo's creation date)[^2]. |
| 3.0 | 2016 | Repackaged around `System.Reactive` with .NET Standard / cross-platform targets. |
| 4.0 | 2018 | Collapsed the split facade packages into a single `System.Reactive` package. |
| 5.0 | 2020 | .NET 5 targets; added `-windows` TFMs (root of later packaging friction). |
| 6.0 | 2023 | Modernization pass; .NET 8 support, updated tooling[^3]. |
| 7.0 | in progress | Planned packaging split to move UI-framework dependencies out of the core[^3]. |

## References

[^1]: ReactiveX — the Reactive Extensions family and the `IObservable<T>`/`IObserver<T>` model.
      http://reactivex.io/
[^2]: dotnet/reactive README — the four libraries (Rx.NET, AsyncRx.NET, Ix.NET, System.Linq.Async),
      .NET Foundation stewardship, and core team. https://github.com/dotnet/reactive
[^3]: Rx.NET packaging and v6/v7 roadmap discussions.
      https://github.com/dotnet/reactive/discussions/2038
[^4]: Introduction to Rx.NET, 2nd Edition (free ebook, based on Lee Campbell's 2010 book).
      https://introtorx.com/
[^5]: Cysharp/R3 — modern Reactive Extensions reimplementation for .NET.
      https://github.com/Cysharp/R3

## Tags

csharp, dotnet, reactive-programming, rx, observable, event-streams, linq, async, functional-reactive, dotnet-foundation
