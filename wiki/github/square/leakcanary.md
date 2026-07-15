# square/leakcanary

> A memory leak detection library for Android that catches retained objects at runtime and prints the exact reference chain keeping them alive.

[GitHub repo](https://github.com/square/leakcanary) ·
[Official website](https://square.github.io/leakcanary) ·
[License: Apache-2.0](https://github.com/square/leakcanary/blob/main/LICENSE.txt)

## Overview

LeakCanary is a debug-time tool from Square, first released in 2015[^1], that
detects memory leaks in Android apps without any manual instrumentation. You add
one dependency; it installs itself, watches objects that are supposed to be
garbage-collected (destroyed Activities, Fragments, Views, ViewModels), and when
one is retained too long it dumps the heap, computes the shortest strong
reference path from a GC root to the leaked object, and shows that "leak trace"
in a notification and a standalone UI. The value is not that it finds leaks —
any heap dump has them — but that it names the specific field-by-field chain a
developer can act on, instead of a raw `.hprof` to spelunk.

The defining tension is that leak detection is expensive and LeakCanary pays for
it at runtime, in the app process. Dumping the heap freezes the app
(stop-the-world) for a noticeable pause, and analysis runs on a background thread
in the same process. That cost is acceptable in a debug build and unacceptable
in production, which is why LeakCanary is designed to ship only in
`debugImplementation` and is explicitly not a production monitoring tool[^2]. The
2.x line (2019) was a full Kotlin rewrite around a new heap analyzer, Shark,
replacing the older HAHA/perflib parser and cutting analysis memory and time
substantially[^3].

## Getting Started

```groovy
// build.gradle — debug builds only; never ship this to release
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.14'
}
```

That is the entire integration. LeakCanary auto-installs via a
`ContentProvider` (`AppWatcher`) that runs on app startup, so there is no code to
call in `Application.onCreate()`. Build and run the debug variant, exercise the
app, and watch for a "LeakCanary" notification when a retained object is found.

```kotlin
// Optional: watch your own objects (e.g. a manually-scoped controller)
AppWatcher.objectWatcher.expectWeaklyReachable(
  myController,
  "MyController should be cleared when the screen closes"
)
```

## Architecture / How It Works

LeakCanary is four cooperating pieces:

- **AppWatcher** — auto-installed on startup. Hooks Android lifecycle callbacks
  and hands destroyed Activities, Fragments, fragment views, ViewModels, and
  services to the ObjectWatcher.
- **ObjectWatcher** — wraps each watched object in a `WeakReference` tied to a
  `ReferenceQueue`. After the object should have been collected, it triggers a
  GC and checks the queue. If the reference was cleared, the object is gone
  (fine). If not after a delay, it becomes a *retained* object.
- **Heap dump + Shark** — once the retained count crosses a threshold (default 5,
  fewer when the app is visible/debuggable), LeakCanary calls
  `Debug.dumpHprofData()` to snapshot the heap, then runs **Shark** — its own
  `.hprof` parser and graph engine — to find the shortest path of strong
  references from GC roots to each retained object.
- **HeapAnalyzerService** — runs the Shark analysis on a background thread and
  renders the resulting leak trace into the notification and the LeakCanary
  activity, grouping identical leaks by signature so the same leak isn't
  reported repeatedly.

The leak trace is the product: a chain like
`GC Root → static Foo.instance → Bar.context → destroyed MainActivity`, with
LeakCanary marking which references it is confident are the culprit (the
"leaking" / "not leaking" annotations) using known heuristics about the Android
framework. Shark ships known-leak fingerprints (`AndroidReferenceMatchers`) so
that well-documented framework leaks are labeled rather than blamed on your code.

Shark is deliberately decoupled: it is a plain JVM library that parses any HProf
file, and there is a `shark-cli` for analyzing dumps off-device. LeakCanary the
Android artifact is essentially the runtime plumbing that decides *when* to hand
a heap to Shark.

## Production Notes

- **Debug-only, by design.** The heap dump is a full stop-the-world pause that
  can last from a fraction of a second to several seconds on large heaps; users
  would feel it. Keep it on `debugImplementation`. If you need any retained-object
  signal in release, use `leakcanary-object-watcher-android` alone (watching,
  no dumping/analysis) rather than the full artifact[^2].
- **Heap dumps scale with app size.** A large heap means a large `.hprof` and a
  longer, more memory-hungry Shark pass. On memory-constrained devices the
  analysis itself can approach OOM; this was the primary motivation for the 2.x
  Shark rewrite, which streams the heap instead of loading it wholesale[^3].
- **False positives and framework leaks.** Many reported traces terminate in
  Android framework objects (InputMethodManager, etc.) that leak regardless of
  your code. LeakCanary tries to label these, but the known-leak list lags new
  Android releases, so triage still requires judgment.
- **CI / instrumentation tests.** LeakCanary can fail instrumentation tests on
  detected leaks via `FailTestOnLeakRunListener`, but heap-dumping inside a test
  suite is slow and flaky under emulators; most teams gate it narrowly rather
  than running it on every test.
- **Config timing.** `LeakCanary.config` can disable dumping, change thresholds,
  or add custom `ObjectInspectors`, but because AppWatcher installs automatically
  you configure it *after* install, not to prevent it.
- **1.x → 2.x is a hard migration.** The 2.0 rewrite changed the package names,
  the API surface, and the analyzer wholesale; there is no drop-in upgrade from
  the old `RefWatcher`-centric 1.x API[^3].

## When to Use / When Not

**Use when:**
- You develop an Android app and want automatic, zero-code leak detection in
  debug builds.
- You have a leak and need the exact reference chain, not just "the heap grew."
- You want a scriptable off-device analyzer (`shark-cli`) for `.hprof` files.

**Avoid when:**
- You need leak/OOM monitoring in production at scale — LeakCanary's in-process,
  stop-the-world dump is the wrong shape; use a forked-process production tool.
- You are chasing *native* (C/C++/JNI) memory growth — LeakCanary analyzes the
  Java/Kotlin object graph only.
- You want a one-time manual investigation of a captured dump — a dedicated heap
  analyzer gives more interactive control.

## Alternatives

- Tencent/matrix — Android APM suite whose ResourceCanary module does similar
  leak detection; use it when you want a broader performance-monitoring platform
  rather than a single-purpose dev tool.
- KwaiAppTeam/KOOM — online OOM/leak monitoring that dumps the heap in a forked
  process; use it when you need leak detection in *release* builds at scale
  without freezing the app.
- google/perfetto — system-wide tracing with native heap profiling (heapprofd);
  use it for native memory and cross-process analysis, not Java object leaks.
- Android Studio Memory Profiler — interactive, IDE-driven heap inspection; use
  it when you want to poke around a live heap manually instead of automated
  leak-trace reports.
- Eclipse MAT — offline `.hprof` analyzer with dominator trees and OQL; use it
  for deep manual forensics on a captured dump.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015 | Initial release; `RefWatcher` API, HAHA/perflib heap parser[^1]. |
| 2.0 | 2019-11 | Full Kotlin rewrite; Shark analyzer, auto-install, new UI[^3]. |
| 2.7 | 2021 | Jetpack Compose support added. |
| 2.14 | 2024 | Recent 2.x maintenance release (latest at time of writing). |

## References

[^1]: LeakCanary announcement, Square Engineering Blog — 2015. https://developer.squareup.com/blog/leakcanary/
[^2]: LeakCanary documentation, "Fundamentals" (debug-only usage, object watching). https://square.github.io/leakcanary/fundamentals/
[^3]: LeakCanary 2.0 release / Shark heap analyzer overview. https://square.github.io/leakcanary/shark/

## Tags

android, kotlin, memory-leak, heap-analysis, debugging, jvm, mobile, profiling, developer-tools, square
