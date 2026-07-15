# oshai/kotlin-logging

> A thin Kotlin logging facade — mostly an slf4j wrapper on the JVM — that trades boilerplate and eager string-building for lambda-based lazy log messages.

[GitHub repo](https://github.com/oshai/kotlin-logging) ·
[License: Apache-2.0](https://github.com/oshai/kotlin-logging/blob/master/LICENSE)

## Overview

kotlin-logging is a logging facade for Kotlin authored by Ohad Shai (`oshai`), first published in 2016[^1]. It exists to answer a recurring question — "what is the idiomatic way to log in Kotlin?" — and its answer is deliberately small: on the JVM it is a wrapper over slf4j-api, adding two things Kotlin developers kept hand-rolling. First, `KotlinLogging.logger {}` derives the logger name from the enclosing class/file, removing the `LoggerFactory.getLogger(Foo::class.java)` boilerplate. Second, all log calls take a lambda (`logger.debug { "..." }`) so the message string is only built when the level is actually enabled, replacing the `if (logger.isDebugEnabled)` guard pattern.

It is not a logging backend. On the JVM it produces no output on its own — you must also supply slf4j-api and a concrete binding (Logback, Log4j2, `slf4j-simple`, etc.). This is the single most common source of confusion for new users, and it became stricter in version 5, which stopped bundling slf4j-api transitively[^3]. The library's value proposition is entirely about ergonomics at the call site, not about routing, formatting, or appenders — those remain the backend's job.

The project is small, single-maintainer, and stable rather than fast-moving; at ~3,100 stars and a last push in mid-2026 it is maintained but not under heavy active development. It has quietly become a default choice in a large slice of the Kotlin/JVM ecosystem — JetBrains YouTrack, Square's Misk, and pact-jvm are among the named users[^2]. Beyond the JVM it offers Kotlin Multiplatform artifacts, but that support is explicitly labeled experimental[^4].

## Getting Started

Add the JVM artifact plus an slf4j binding (v5+ requires you to bring both slf4j-api and a backend):

```groovy
// build.gradle
implementation 'io.github.oshai:kotlin-logging-jvm:7.0.3'
implementation 'org.slf4j:slf4j-simple:2.0.13'   // any slf4j binding
```

```kotlin
import io.github.oshai.kotlinlogging.KotlinLogging

// Declared above the class -> compiled as a file-level (effectively static) logger
private val logger = KotlinLogging.logger {}

class FooWithLogging {
    fun bar() {
        val expensive = "world"
        logger.debug { "hello $expensive" }        // lambda not evaluated if DEBUG is off
        logger.error(RuntimeException("boom")) { "failed for $expensive" }
    }
}
```

## Architecture / How It Works

The core abstraction is `KLogger`, obtained through `KotlinLogging.logger {}`. The empty lambda is a trick: it is a no-op closure whose *enclosing class* is inspected via reflection (on the JVM, the lambda's declaring class) to derive the logger name. Place the declaration at file scope, above the class, and the compiler emits it as a static field on the file's synthetic class; place it inside the class body and you get one logger reference per instance. The name-derivation is why the empty `{}` matters — passing a name string is also supported when you want to override it.

On the JVM, `KLogger` delegates to an slf4j `Logger`. Every idiom maps down to a plain slf4j call: `logger.debug { msg }` compiles to the `isDebugEnabled` guard plus `logger.debug(msg())`, and markers, MDC, parameterized messages, and location awareness all pass through to slf4j. Because it is a pure facade, any slf4j implementation and any slf4j-consuming tooling (log aggregators, bridges like `jul-to-slf4j`) continue to work unchanged. The escape hatch `DelegatingKLogger.underlyingLogger` exposes the wrapped backend logger when you need implementation-specific APIs.

Two newer surfaces sit on top of this. A fluent builder — `logger.atWarn { message = ...; cause = ...; payload = ... }` — mirrors slf4j 2.x's fluent logging API and lets you attach structured key/value payloads. And the multiplatform build splits the code into `commonMain` plus per-target source sets: the JVM target binds to slf4j, while JS and Native targets use simpler built-in outputs. The common API is intentionally a subset, which is why multiplatform is still called experimental — the class hierarchy differs enough between targets that some slf4j-specific behavior has no equivalent off the JVM[^4].

## Production Notes

**No binding, no logs.** The most frequent failure mode is silence: kotlin-logging without an slf4j binding on the classpath produces nothing (or slf4j's "no providers were found" warning). In v5+ you must also declare slf4j-api yourself — a transitive dependency you may have relied on in v1–v4 that is no longer pulled in[^3].

**The v3/v4 → v5 migration is a hard break.** Version 5 changed the Maven group id from `io.github.microutils` to `io.github.oshai` and the root package from `mu` to `io.github.oshai.kotlinlogging`[^3]. This means every `import mu.KotlinLogging` in your codebase has to be rewritten, and coordinate strings in build files must change. The upside: old and new versions can coexist on one classpath (different packages, different coordinates), so libraries on the old version and application code on the new one do not conflict — you can migrate incrementally.

**Lambda cost is real but small.** The lazy-message lambda avoids string construction when a level is disabled, but a capturing lambda still allocates. In genuinely hot loops logging at a disabled level millions of times per second, the allocation and megamorphic call can show up in a profiler; for ordinary application logging it is negligible and strictly cheaper than eager `String.format`.

**Logger-name surprises.** Because the name comes from the lambda's enclosing class, defining the logger in an unexpected scope (inside a companion, a top-level function, or a lambda passed elsewhere) can yield a logger named after a synthetic or anonymous class. When log filtering by category matters, verify the emitted name rather than assuming it.

**Level configuration is not here.** Setting levels, formats, and appenders is done entirely in the backend's config (`logback.xml`, `log4j2.xml`, JVM system properties for `slf4j-simple`). A common "my debug logs don't show" report is really a backend-configuration issue, not a kotlin-logging one.

## When to Use / When Not

**Use when:**
- You are on Kotlin/JVM and already committed to the slf4j ecosystem (Logback/Log4j2) and want the Kotlin sugar over it.
- You want lazy, lambda-evaluated log messages without writing `isXEnabled` guards.
- You want logger declarations without repeating the class name.
- You need to interoperate with existing Java/slf4j logging config and tooling untouched.

**Avoid when:**
- You want a self-contained logger: this is a facade and needs a backend wired up.
- You are building pure Kotlin Multiplatform (mobile/Native) and need first-class, non-experimental multiplatform logging — a KMP-native library fits better.
- You want structured/coroutine-aware logging as a first-class model rather than an slf4j-shaped one.
- You are avoiding reflection-based logger-name derivation for startup-time or GraalVM-native reasons.

## Alternatives

- qos-ch/slf4j — the facade kotlin-logging wraps; use it directly (with `LoggerFactory`) when you do not want the Kotlin conveniences or the extra dependency.
- qos-ch/logback — a backend, not a facade; you generally pair it *with* kotlin-logging, but it also has its own Kotlin-friendly usage if you skip the wrapper.
- kloggingorg/klogging — pure-Kotlin, coroutine-aware, structured logging that is not an slf4j facade; use when you want native Kotlin logging and structured events rather than slf4j compatibility.
- touchlab/Kermit — Kotlin Multiplatform logging with pluggable writers and crash-reporting integrations; use for KMP mobile apps.
- AAkira/Napier — lightweight KMP logger for Android/iOS/JS; use when you want minimal multiplatform logging without the slf4j model.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | Initial release under `MicroUtils/kotlin-logging`, package `mu`[^1]. |
| 1.4 | ~2017 | Location awareness (caller info) added[^2]. |
| 2.x–3.x | 2018–2022 | slf4j facade matures; multiplatform artifacts introduced (experimental)[^4]. |
| 5.0 | 2023 | Breaking: group id `io.github.microutils` → `io.github.oshai`, package `mu` → `io.github.oshai.kotlinlogging`; slf4j-api no longer bundled[^3]. |
| 7.0.3 | 2024–2025 | Current line; fluent `atWarn {}` API with structured payloads, ongoing multiplatform work[^5]. |

## References

[^1]: Ohad Shai, "No forks, one star, now what — how I published my open source projects." https://medium.com/@OhadShai/no-forks-one-star-now-what-how-i-published-my-open-source-projects-8a5b5ae35d2c
[^2]: kotlin-logging README, "Who is using it" and FAQ sections. https://github.com/oshai/kotlin-logging/blob/master/README.md
[^3]: Version 5 migration notes and issue #264. https://github.com/oshai/kotlin-logging/issues/264
[^4]: kotlin-logging wiki, "Multiplatform support." https://github.com/oshai/kotlin-logging/wiki/Multiplatform-support
[^5]: kotlin-logging ChangeLog. https://github.com/oshai/kotlin-logging/blob/master/ChangeLog.md

## Tags

kotlin, logging, slf4j, jvm, facade, multiplatform, android, lazy-evaluation, apache-2.0, structured-logging
