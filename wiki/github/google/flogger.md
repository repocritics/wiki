# google/flogger

> Google's fluent logging API for Java, designed so that disabled log statements cost effectively nothing.

[GitHub repo](https://github.com/google/flogger) ·
[Official website](https://google.github.io/flogger/) ·
[License: Apache-2.0](https://github.com/google/flogger/blob/master/LICENSE)

## Overview

Flogger is a logging *API* — a front-end that application code calls — rather than a logging *backend* that writes records to files or sockets. It was designed and implemented by David Beaumont with the Java Core Libraries Team (the Guava maintainers) to unify the many incompatible logging front-ends that had accumulated across Google's monorepo, and was open-sourced in 2018[^1]. It is now Google's sole recommended internal Java logging API.

The defining idea is the **fluent call chain**: instead of `logger.info(String)`, you write `logger.atInfo().withCause(e).log("...")`. The first call, `atInfo()`, returns a logging context whose *type depends on whether the level is enabled*. When the level is disabled, it returns a shared no-op singleton, so the entire rest of the chain — argument boxing, string formatting, the varargs array — is skipped and nothing is allocated[^2]. This is what lets Flogger claim that fine-grained log statements are "effectively free" when turned off, and it is the concrete engineering argument the project rests on, not fluency as a style preference.

The tradeoff is ecosystem size and integration friction. Flogger is a front-end only; it must be wired to a backend (JDK `java.util.logging` by default, or SLF4J/Log4j via adapter modules), and outside Google its adoption, tooling, and Stack Overflow surface area are a fraction of SLF4J's. Teams adopt it for the disabled-logging performance and the built-in rate limiting, and pay for it in a smaller community and an extra configuration step.

## Getting Started

Maven — you need both the API and a backend at runtime:

```xml
<dependency>
  <groupId>com.google.flogger</groupId>
  <artifactId>flogger</artifactId>
  <version>0.8</version>
</dependency>
<dependency>
  <groupId>com.google.flogger</groupId>
  <artifactId>flogger-system-backend</artifactId>
  <version>0.8</version>
  <scope>runtime</scope>
</dependency>
```

```java
import com.google.common.flogger.FluentLogger;

class MyClass {
  private static final FluentLogger logger = FluentLogger.forEnclosingClass();

  void handle(Request req, Exception e) {
    logger.atInfo().log("Handling request %s", req.id());

    logger.atWarning()
        .withCause(e)
        .atMostEvery(30, SECONDS)          // rate-limit this log site
        .log("Retry failed for %s", req.id());
  }
}
```

Note the format specifiers are Java `printf` style (`%s`, `%d`, `%016x`), **not** the SLF4J `{}` brace form[^1]. This is the single most common surprise for developers arriving from SLF4J/Logback.

## Architecture / How It Works

The public surface is small; almost all machinery is in the fluent chain:

1. **`FluentLogger.forEnclosingClass()`** — creates a logger keyed to the calling class. It infers the class name (historically via a stack-trace lookup on platforms without a cheaper mechanism), so the recommended pattern is always `private static final` — one logger per class, resolved once at class initialization.
2. **`atInfo()` / `atWarning()` / `atSevere()` …** — return a `LoggingApi` context. If the backend reports the level disabled, this is a stateless `NoOp` singleton and the chain short-circuits[^2]. If enabled, it is a mutable `LogContext` that accumulates metadata.
3. **Chained modifiers** — `withCause()`, `with(key, value)` (typed `MetadataKey`s), `atMostEvery()`, `every(n)`, `per(...)` bucketing — all mutate the context and return `this`.
4. **`log("fmt", args…)`** — the terminal operation. Flogger provides many `log` overloads with fixed arities (`log(String, Object)`, `log(String, Object, Object)`, …) plus primitive specializations, specifically to avoid allocating a varargs `Object[]` and boxing primitives on the enabled path.
5. **Backend** — the `LoggerBackend` abstraction actually emits the record. `flogger-system-backend` targets `java.util.logging`; `flogger-slf4j-backend`, `flogger-log4j-backend`, and `flogger-log4j2-backend` bridge to those systems. The API artifact is backend-agnostic; the backend is a separate runtime dependency.

`GoogleLogger` is a near-identical variant tuned for Google's internal codebase; the project explicitly recommends `FluentLogger` for external code because its API is held more stable[^1]. Deferred/expensive arguments use `LazyArgs.lazy(() -> ...)` so the supplier only runs when the statement is actually logged.

## Production Notes

**Rate-limiting state is per-log-site.** `atMostEvery(30, SECONDS)` and `every(100)` keep their counters keyed to the physical source line, shared across all threads hitting it. This is usually what you want, but it means two different messages on the same conceptual event throttle independently, and a single line in a hot loop shares one budget across all callers. Use `per(enum/key, strategy)` to bucket the limit by a dimension (e.g. per-tenant).

**You must ship a backend, and configuring it is the awkward step.** With `flogger-system-backend` the output goes through `java.util.logging`, whose configuration (`logging.properties`, handlers, levels) is notoriously unfriendly and separate from Flogger itself. Most non-trivial deployments switch to the SLF4J or Log4j2 backend so Flogger output flows through the same pipeline as the rest of the dependency tree. Only one backend should be on the runtime classpath.

**The disabled-logging win depends on discipline.** The "free when disabled" property holds only if expensive work sits *after* `atX()` in the chain (as a `%s` argument or `lazy(...)`), not computed eagerly and passed in. `logger.atFine().log("%s", buildBigDebugString())` still builds the string every time; `lazy(() -> buildBigDebugString())` does not.

**Static analysis / lint.** Google ships an Error Prone check set for Flogger misuse (non-`static`/`final` loggers, format-string/argument mismatches, `%s` vs `{}` confusion). Outside a Google-style build with Error Prone wired in, these mistakes are silent until runtime.

**Versioning and maturity.** Flogger has stayed on `0.x` versions on Maven Central for its entire public life, with infrequent releases[^3]. In practice the API is stable and widely used inside Google, but the perpetual `0.x` and low release cadence read as under-maintained to teams evaluating it purely from the outside. Interop is fine — it coexists with SLF4J/Log4j through the backends — but there is no large third-party plugin ecosystem of its own.

## When to Use / When Not

**Use when:**
- You have many fine-grained `atFine()`/`atFinest()` statements you want to leave in permanently and pay near-zero cost when disabled.
- You want built-in rate limiting (`atMostEvery`, `every`, `per`) without a homegrown throttle.
- You value the self-documenting fluent call sites and `printf` formatting.
- You are already in a Google-style Java toolchain (Guava, Error Prone) where the lint support comes for free.

**Avoid when:**
- Your team and dependencies are deeply invested in SLF4J's `{}` idiom and its huge ecosystem; the switching cost rarely pays back.
- You want a single logging dependency with no separate backend wiring step.
- You need a large community, many appenders, and heavy third-party integration out of the box — Log4j2 and Logback win there.
- Perpetual `0.x` versioning is a procurement or governance blocker for you.

## Alternatives

- qos-ch/slf4j — use instead when you want the de facto Java logging façade with the widest backend and library support.
- qos-ch/logback — use instead when you want a complete, batteries-included logging implementation behind SLF4J, not just an API.
- apache/logging-log4j2 — use instead when you need async loggers, high-throughput appenders, and a large plugin ecosystem.
- tinylog-org/tinylog — use instead when you want a lightweight logger with a similarly lean API and minimal footprint.
- google/guava — same maintainers; not a logger, but the toolchain and design sensibility Flogger comes from.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-04 | Open-sourced by Google; repo created[^1]. |
| 0.4 | 2019 | Early public releases on Maven Central under `com.google.flogger`. |
| 0.6–0.7.x | 2020–2022 | Backend adapters (SLF4J, Log4j, Log4j2), metadata/rate-limit refinements[^3]. |
| 0.8 | 2023 | Latest line at time of writing; API remains `0.x`[^3]. |

## References

[^1]: Flogger README and homepage — "A Fluent Logging API for Java", design by David Beaumont with the Java Core Libraries Team. https://google.github.io/flogger/
[^2]: Flogger — "Benefits: cheap disabled logging." The disabled path returns a no-op context so the chain does no work. https://google.github.io/flogger/benefits#cheap-disabled-logging
[^3]: Flogger releases on Maven Central, group `com.google.flogger`. https://search.maven.org/search?q=g:com.google.flogger

## Tags

java, logging, logging-api, fluent-interface, google, jvm, observability, performance, structured-logging, backend-agnostic
