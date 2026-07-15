# JakeWharton/timber

> A thin logging facade for Android that adds automatic tags and pluggable output "trees" on top of `android.util.Log`.

[GitHub repo](https://github.com/JakeWharton/timber) ·
[Documentation](https://jakewharton.github.io/timber/docs/5.x/) ·
[License: Apache-2.0](https://github.com/JakeWharton/timber/blob/trunk/LICENSE.txt)

## Overview

Timber is a small Android logging utility by Jake Wharton, extracted in 2013 from a
`Log` wrapper class he was hand-copying into every app he built[^1]. The pitch is
deliberately modest: it is not a logging framework in the SLF4J/Logback sense, it is a
static facade (`Timber.d`, `Timber.e`, …) that forwards each call to a list of installed
`Tree` instances. The library ships roughly a dozen source files and no transitive
dependencies beyond the Android SDK.

The defining design choice is that **nothing is logged until you plant a `Tree`**. There
is no default output — the README's line is "every time you log in production, a puppy
dies." In development you plant a `DebugTree`, which infers the log tag from the calling
class automatically; in release builds you either plant nothing (silent) or plant a custom
`Tree` that forwards crashes to Crashlytics/Sentry and drops verbose/debug noise. This
"one API surface, swap the sink per build type" model is the whole value proposition.

Timber is effectively feature-complete. The current stable release, 5.0.1, dates to 2021,
and the `trunk` branch still receives occasional maintenance commits (dependency and
toolchain bumps) rather than new features[^2]. For a facade this thin that is a reasonable
steady state, not abandonment — but callers expecting active feature development should
calibrate accordingly.

## Getting Started

```groovy
// build.gradle
dependencies {
  implementation 'com.jakewharton.timber:timber:5.0.1'
}
```

```kotlin
// Application.onCreate — plant early, once.
class App : Application() {
  override fun onCreate() {
    super.onCreate()
    if (BuildConfig.DEBUG) {
      Timber.plant(Timber.DebugTree())
    } else {
      Timber.plant(CrashReportingTree())   // your own Tree
    }
  }
}

// Anywhere in the app — no tag needed, DebugTree derives it.
Timber.d("User %s logged in", user.id)
Timber.e(throwable, "Upload failed")
Timber.tag("Network").v("request sent")   // explicit one-shot tag
```

A custom production `Tree` overrides a single method:

```kotlin
class CrashReportingTree : Timber.Tree() {
  override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
    if (priority == Log.VERBOSE || priority == Log.DEBUG) return
    Crashlytics.log(priority, tag, message)
    if (t != null) Crashlytics.recordException(t)
  }
}
```

## Architecture / How It Works

`Timber` is a Kotlin `object` (as of 5.0) that holds an array of planted `Tree`s. The
public static-style methods (`v`/`d`/`i`/`w`/`e`/`wtf`) iterate that array and call each
tree's `log(priority, tag, message, t)`. Trees are added with `plant`, removed with
`uproot`/`uprootAll`, and the planted set is exposed via `forest()`. For hot-path
performance Timber keeps a plain `Array<Tree>` snapshot (`treeArray`) alongside the backing
list so the log loop avoids iterator allocation.

`Tree` is an abstract base. It implements the priority methods and the format-string
handling (message formatting is done once, lazily, and only if at least one tree will
consume it), then delegates the actual sink to the abstract `log(...)`. `DebugTree`
subclasses it and implements two behaviors: it forwards to `android.util.Log`, and it
derives the tag from the current stack trace — reading the class name of the frame that
called into Timber and stripping anonymous-class suffixes.

The tag-inference path is the interesting coupling. Because the tag comes from a
`Throwable().stackTrace` lookup at a fixed depth, `DebugTree` is sensitive to how many
frames sit between your call site and the tree. Wrapping Timber in your own helper, or
calling it through inline/lambda layers, shifts that frame and produces the wrong tag.
This is why the docs tell you to call Timber's static methods directly rather than build a
façade over the façade.

Timber also ships **Android Lint rules** in the artifact (`TimberArgCount`,
`TimberArgTypes`, `TimberTagLength`, `LogNotTimber`, `StringFormatInTimber`,
`BinaryOperationInTimber`, `TimberExceptionLogging`). These catch format-string arity/type
mismatches, tags over Android's 23-char limit, raw `Log.*` usage, and redundant
`String.format`/concatenation inside Timber calls at compile time — a genuinely useful part
of the package that is easy to overlook.

## Production Notes

- **`DebugTree` is a debug tool, not a production sink.** Its stack-trace tag inference is
  comparatively expensive and, more importantly, meaningless after R8/ProGuard renames
  classes. Do not plant `DebugTree` in release builds; write a `Tree` that hard-codes or
  omits tags and routes to your crash reporter.
- **Kotlin string interpolation defeats lazy logging.** `Timber.d("$expensiveToString")`
  builds the string at the call site regardless of whether any tree consumes it, because
  the argument is evaluated before Timber is entered. Use Timber's printf-style
  `Timber.d("value=%s", lazy)` form, or guard the call, when the message is costly. The
  library only skips *formatting* work when no tree is planted — it cannot skip work the
  caller already did.
- **Forgetting to plant is silent.** With no tree installed every log call is a no-op and
  no warning is emitted. New engineers frequently "lose" their logs this way.
- **Tag inference breaks through wrappers.** Any indirection between the call site and the
  tree corrupts the auto-derived tag (see Architecture). If you must centralize logging,
  set tags explicitly with `Timber.tag(...)` rather than relying on inference.
- **5.0 was a Kotlin rewrite.** Version 5.0.0 (2021) migrated the source from Java to
  Kotlin and adjusted nullability on the `Tree.log` signature; most Java callers were
  unaffected, but subclasses overriding `log` occasionally needed nullability tweaks[^2].
- **Release cadence is slow.** Depending on `5.0.1` is safe and widely done, but do not
  expect frequent updates; a `5.1.0-SNAPSHOT` has existed in Sonatype snapshots without a
  corresponding stable cut.

## When to Use / When Not

**Use when:**
- You have an Android app and want structured, taggable logging with a clean per-build-type
  sink split (debug console vs. production crash reporter).
- You want compile-time lint checks on your log statements.
- You want a dependency-free, tiny, stable facade you will not have to think about again.

**Avoid when:**
- You are on the JVM/server or Kotlin Multiplatform — Timber is Android-only and built on
  `android.util.Log`. Use SLF4J or a KMP logger instead.
- You need structured/JSON logging, log levels configurable at runtime, appenders, MDC, or
  rolling files — that is SLF4J/Logback territory, not Timber's.
- You want zero-overhead lazy logging via lambdas as a first-class API; Timber's printf
  model works but a lambda-based logger expresses it more directly.

## Alternatives

- square/logcat — Square's Kotlin Android logger; lambda-based lazy messages and no
  stack-trace tag cost. Use instead when the tag-inference footgun bothers you.
- oshai/kotlin-logging — SLF4J-backed Kotlin facade. Use when you need the same logging
  code on Android and the JVM/server.
- touchlab/Kermit — Kotlin Multiplatform logger. Use when your log calls live in shared
  KMP code targeting Android + iOS.
- AAkira/Napier — another KMP logging facade with a Timber-like API. Use for multiplatform
  projects wanting a familiar surface.
- qos-ch/slf4j — the JVM standard with logback-android backends. Use when you need
  appenders, structured output, or runtime configuration Timber does not provide.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2013-07 | Initial extraction from Jake Wharton's copied `Log` wrapper[^1]. |
| 4.0.0 | 2015-09 | API reshaped around `Tree`/`plant`; explicit `tag(...)` chaining. |
| 4.7.1 | 2018 | Last 4.x (Java) release; embedded Lint rules matured. |
| 5.0.0 | 2021-08 | Rewritten in Kotlin; `Timber` becomes a Kotlin `object`[^2]. |
| 5.0.1 | 2021 | Current stable; bugfix over 5.0.0. |

## References

[^1]: Timber README — origin note ("I copy this class into all the little apps I make…"),
Apache-2.0, Jake Wharton, 2013. https://github.com/JakeWharton/timber
[^2]: Timber releases and changelog. https://github.com/JakeWharton/timber/releases

## Tags

android, kotlin, logging, logger, log, mobile, facade, jvm, library, apache-2.0
