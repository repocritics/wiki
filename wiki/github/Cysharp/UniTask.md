# Cysharp/UniTask

> Zero-allocation async/await for Unity, built on the engine's PlayerLoop instead of threads.

[GitHub repo](https://github.com/Cysharp/UniTask) ·
[License: MIT](https://github.com/Cysharp/UniTask/blob/master/LICENSE)

## Overview

UniTask is Cysharp's replacement for `System.Threading.Tasks.Task` inside Unity. The
core objection it answers is that .NET's `Task<T>` is a heap-allocated reference type
built around a thread pool and `SynchronizationContext` — a model that fits poorly with
Unity, which is single-threaded at the gameplay layer and dispatches its own asynchronous
work (`AsyncOperation`, coroutines) from the engine's PlayerLoop[^1]. UniTask replaces
`Task<T>` with a struct `UniTask<T>` plus a custom async method builder and a pooled
`IUniTaskSource`, so an `async` method that never actually suspends allocates nothing[^2].

Because it runs entirely on the PlayerLoop and uses no threads or `SynchronizationContext`
by default, UniTask works on platforms where the .NET thread pool is unavailable or
restricted — notably WebGL and other wasm targets. That same design is the source of its
sharpest tradeoff: a `UniTask` is a single-consumer value, closer to `ValueTask` than
`Task`, and awaiting one twice is undefined behavior. Teams coming from `Task` bring habits
(cache a task in a field, await it from several callers, `.Result` it) that are silently
wrong here.

Cysharp — the studio behind MessagePack-CSharp, MagicOnion, ZString and R3 — maintains it,
and UniTask has effectively become the default async library for serious Unity projects.
As of this writing the repo has 11,051 stars and 1,011 forks, with commits landing through
mid-2026, so it is actively maintained rather than merely popular.

## Getting Started

Install via UPM with a git reference (add to `Packages/manifest.json` or use the Package
Manager "Add package from git URL"):

```
https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask
```

A `.unitypackage` is also attached to each GitHub release. Minimum supported engine is
Unity 2018.4 (UniTask relies on the C# 7 task-like builder feature)[^1].

```csharp
using Cysharp.Threading.Tasks;
using UnityEngine;
using UnityEngine.Networking;

async UniTask<string> LoadAsync(CancellationToken ct)
{
    // Unity AsyncOperations become awaitable
    var asset = await Resources.LoadAsync<TextAsset>("foo").WithCancellation(ct);

    var req = await UnityWebRequest.Get("https://example.com").SendWebRequest()
                                   .WithCancellation(ct);

    await UniTask.Delay(TimeSpan.FromSeconds(1), cancellationToken: ct); // no thread timer
    return req.downloadHandler.text;
}
```

Pass `this.GetCancellationTokenOnDestroy()` (or `destroyCancellationToken` on Unity 2022.2+)
as the token so pending awaits abort when the GameObject is destroyed.

## Architecture / How It Works

The defining piece is the custom async method builder. When the C# compiler lowers an
`async UniTask<T>` method, it uses `AsyncUniTaskMethodBuilder`, which pulls its state
machine box from a `TaskPool` rather than allocating. A completed `UniTask` carries its
result inline in the struct; only a genuinely suspended one rents an `IUniTaskSource`.
This is why the README's profiler screenshots show zero GC for hot-path awaits — the cost
was moved into a reusable pool, not eliminated by magic.

Timing primitives (`UniTask.Yield`, `Delay`, `DelayFrame`, `WaitUntil`, `NextFrame`,
`WaitForEndOfFrame`) are implemented as items injected into Unity's PlayerLoop at fixed
`PlayerLoopTiming` points (Initialization, Update, PreLateUpdate, and so on). UniTask
rewrites the PlayerLoop at startup to insert its runners; on a normal player this happens
via `[RuntimeInitializeOnLoadMethod]`. There are no background threads involved unless you
explicitly call `UniTask.SwitchToThreadPool` and switch back with `SwitchToMainThread`.

Around this core sit several layers: `UniTaskCompletionSource` (a `TaskCompletionSource`
analog for bridging callbacks), `UniTaskAsyncEnumerable` with an async LINQ operator set,
`Channel`, and `AsyncReactiveProperty` for push-based state. A `UniTaskTracker` editor
window lists live UniTasks so leaked/never-completing awaits are visible. Compatibility with
`Task`/`ValueTask` is deliberate: conversions exist (`AsUniTask`, `AsTask`,
`ToUniTask` on `IEnumerator`), and the value-type semantics mirror `IValueTaskSource`.

Since Unity 2023.1 the engine ships its own `Awaitable` type covering some of the same
ground. UniTask remains broader (async LINQ, richer timing, tracker, wider version support)
but the two now overlap, and UniTask documents the comparison directly[^3].

## Production Notes

- **Await-once is a real footgun.** `UniTask` and `UniTask<T>` are single-consumer. Storing
  one in a field and awaiting it from multiple places, awaiting the same variable twice, or
  touching `.GetAwaiter().GetResult()` on an incomplete task is undefined behavior — the
  pooled source may already have been recycled for another task. Use `.Preserve()` or
  `UniTask.Lazy` when you genuinely need multiple awaits; use `UniTaskCompletionSource` when
  many callers must await one event.

- **Cancellation is PlayerLoop-polled by default.** PlayerLoop-based operations
  (`Yield`, `Delay`, …) check the `CancellationToken` on the next loop tick, so cancellation
  is not instantaneous. `cancelImmediately: true` makes it prompt but registers a callback
  via `CancellationToken.Register`, which is measurably heavier — do not default to it.

- **`OperationCanceledException` is the cancellation signal, and throwing it is not free.**
  On hot paths that cancel frequently, `UniTask.SuppressCancellationThrow()` returns
  `(bool IsCanceled, T Result)` instead of throwing. It only suppresses when called on the
  source method directly.

- **Unhandled exceptions vanish into a global sink.** Anything not caught in an async method
  propagates to `UniTaskScheduler.UnobservedTaskException` (logged by default);
  `OperationCanceledException` is silently ignored there. Fire-and-forget work started with
  `.Forget()` / `async UniTaskVoid` will not surface failures unless you observe them.

- **Timeouts should be tokens, not `.Timeout()`.** `.Timeout()`/`.TimeoutWithoutException()`
  act from outside the task and cannot stop the underlying work — they only discard the
  result. Prefer `CancellationTokenSource.CancelAfterSlim(TimeSpan)` (PlayerLoop-based; the
  standard `CancelAfter` uses a threading timer you should avoid in Unity) and pass the token
  in. `TimeoutController` reuses the CTS to avoid per-call allocation.

- **Editor and unit-test contexts differ from the player.** The PlayerLoop injection and
  timing behavior are not identical under the Test Runner / edit mode; `UniTask.Yield` and
  friends require the runtime loop. UniTask ships helpers and a `UniTaskSynchronizationContext`
  for these cases, but expect setup friction when testing async code.

## When to Use / When Not

**Use when:**
- You write `async`/`await` against Unity `AsyncOperation`s, web requests, addressables, or
  frame-based waits and want coroutines gone.
- Allocation/GC pressure matters (mobile, VR) and you are on the per-frame hot path.
- You target WebGL/wasm or otherwise cannot rely on the thread pool.
- You want async LINQ over event streams or a leak-visibility tool (the tracker window).

**Avoid when:**
- Your team cannot internalize the await-once / single-consumer contract — misuse produces
  intermittent, hard-to-reproduce corruption rather than clean errors.
- You can target Unity 2023.1+, want no third-party dependency, and the built-in `Awaitable`
  covers your needs.
- You need genuine CPU parallelism; UniTask is a main-thread orchestration tool, not a
  parallel-compute framework (use jobs/threads for that and marshal results back).

## Alternatives

- Unity Awaitable (built-in, `UnityEngine.Awaitable`, since 2023.1) — use when you can drop
  older-Unity support and want async without an external package, accepting a narrower API.
- Cysharp/R3 — same authors; reactive streams (Rx) rather than one-shot awaits. Use when
  your problem is "values over time / event composition," not "await this operation once."
- neuecc/UniRx — the older reactive library from the same author, now largely superseded by
  R3; still widespread in legacy projects.
- dotnet/runtime `System.Threading.Tasks` — standard `Task`; correct for off-loop/server-side
  code and true multithreading, but heap-allocating and awkward on Unity's main thread.
- svermeulen/Unity3dAsyncAwaitUtil — minimal await bridge for AsyncOperations/coroutines; use
  when you only need `await someAsyncOperation` and not the zero-alloc machinery or LINQ.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2019 | Initial release: async/await for Unity, awaitable `AsyncOperation`s[^1]. |
| 2.0 | 2020 | Struct-based zero-allocation rewrite; custom method builder + task pool; `UniTaskAsyncEnumerable` / async LINQ; `UniTaskTracker`[^2]. |
| 2.x | 2021–2026 | Ongoing 2.x line: `cancelImmediately`, `WhenEach`, Unity 2022/2023 token & `Awaitable` interop, pooling tuning. |
| — | 2026-07-08 | Latest commit to `master` (per GitHub). |

## References

[^1]: UniTask README — "Basics of UniTask and AsyncOperation" (minimum Unity version, PlayerLoop rationale). https://github.com/Cysharp/UniTask#basics-of-unitask-and-asyncoperation
[^2]: Yoshifumi Kawai (neuecc), "UniTask v2 — Zero Allocation async/await for Unity, with Asynchronous LINQ" (2020). https://medium.com/@neuecc/unitask-v2-zero-allocation-async-await-for-unity-with-asynchronous-linq-1aa9c96aa7dd
[^3]: UniTask README — "vs Awaitable" section. https://github.com/Cysharp/UniTask#vs-awaitable

## Tags

csharp, unity, async-await, game-development, coroutines, zero-allocation, performance, dotnet, playerloop, cancellation
