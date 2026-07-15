# qos-ch/slf4j

> A logging facade for Java: you compile against one API and pick the actual logging backend at deployment time.

[GitHub repo](https://github.com/qos-ch/slf4j) ·
[Official website](http://www.slf4j.org) ·
[License: MIT](https://github.com/qos-ch/slf4j/blob/master/LICENSE.txt)

## Overview

SLF4J (Simple Logging Facade for Java) is an abstraction layer that sits
between application code and a concrete logging framework. Libraries and
applications depend only on `slf4j-api`; the running program chooses a
backend — logback, Log4j 2, reload4j, `java.util.logging`, and others — by
putting the matching "provider" JAR on the classpath[^1]. It was written by
Ceki Gülcü, the original author of Log4j 1.x and of Logback, and is
maintained under his company QOS.ch[^2].

The reason SLF4J exists is a coordination problem: a Java application pulls
in dozens of libraries, each of which historically logged through whatever
framework its author liked. SLF4J lets a library log without imposing a
backend on its consumers, and lets the final application impose exactly one.
`slf4j-api` is consequently one of the most widely depended-upon artifacts
on Maven Central — it is a transitive dependency of a large fraction of the
Java ecosystem, which is also why version skew in it is a recurring source
of pain.

The defining tradeoff is indirection. SLF4J adds a thin layer whose entire
job is to be invisible when configured correctly and cryptic when not. Its
best-known failure modes — "no backend found, all logging silently
discarded" and "multiple backends found" — come directly from the
plug-in-at-deploy-time design.

## Getting Started

Maven — pull the API plus exactly one backend (here, Logback):

```xml
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>2.0.17</version>
</dependency>
<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>  <!-- pulls slf4j-api transitively -->
  <version>1.5.18</version>
</dependency>
```

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Service {
    private static final Logger log = LoggerFactory.getLogger(Service.class);

    void handle(String userId, int items) {
        // Parameterized message: no string concatenation happens
        // unless DEBUG is actually enabled.
        log.debug("user {} submitted {} items", userId, items);
        try {
            // ...
        } catch (Exception e) {
            log.error("processing failed for user {}", userId, e); // last arg = throwable
        }
    }
}
```

The `{}` placeholders are the central ergonomic feature: argument formatting
is deferred until the level is confirmed active, avoiding the classic
`if (log.isDebugEnabled())` guard.

## Architecture / How It Works

SLF4J is a compile-time API plus a runtime lookup. `LoggerFactory` locates a
single backend at startup and every `Logger` call delegates to it.

**Binding vs. provider — the 1.x/2.x split.** In SLF4J 1.x, `LoggerFactory`
looked for the class `org.slf4j.impl.StaticLoggerBinder` on the classpath;
each backend shipped its own copy. This "static binding" is why a missing
backend produced `Failed to load class
org.slf4j.impl.StaticLoggerBinder` followed by a no-op logger[^3]. SLF4J
2.0 replaced this with the `java.util.ServiceLoader` mechanism: backends now
register an `org.slf4j.spi.SLF4JServiceProvider` under `META-INF/services`,
and the old `StaticLoggerBinder` class was removed[^4]. This is the single
most important thing to understand when mixing versions — a 1.x-era backend
JAR will not be discovered by a 2.x `slf4j-api`, and vice versa.

**Bridges route foreign APIs back into SLF4J.** Three adapter JARs —
`jcl-over-slf4j` (Apache Commons Logging), `log4j-over-slf4j` (Log4j 1.x
API), and `jul-to-slf4j` (`java.util.logging`) — reimplement another
logging API on top of SLF4J so that a mixed dependency tree funnels into one
backend. The hard rule: never place a bridge and the corresponding native
backend on the same classpath. `log4j-over-slf4j` together with
`slf4j-log4j12` (or `log4j-over-slf4j` next to a real Log4j) creates a
delegation loop that ends in `StackOverflowError`[^5].

**MDC** (Mapped Diagnostic Context) is a thread-local key/value map that
backends can weave into log output — the standard way to attach a request or
trace ID to every line within a thread. Because it is thread-local, it does
not automatically propagate across thread-pool handoffs or reactive
pipelines; that propagation is the caller's responsibility.

**Markers** and, since 2.0, a **fluent event-builder API**
(`log.atInfo().addKeyValue(...).log(...)`) round out the surface, but the
`Logger` interface with parameterized messages remains what nearly all code
uses.

## Production Notes

- **Silent logging is the number-one footgun.** With no provider on the
  classpath, SLF4J emits one warning to `stderr` at startup and then
  discards every message. In a container with noisy startup output this is
  easy to miss and looks like "logging is broken." Always confirm exactly
  one provider is resolved.
- **Multiple bindings.** More than one backend produces
  `Class path contains multiple SLF4J providers` (1.x: `bindings`), then
  picks one non-deterministically based on classpath order. This surfaces
  constantly through transitive dependencies — two libraries each dragging
  in a different backend. `mvn dependency:tree` is the standard triage tool;
  exclude the unwanted providers.
- **The 1.7 → 2.0 migration is not drop-in for backends.** `slf4j-api` 2.0
  is API-compatible for calling code, but the provider lookup changed
  (above). Any pinned 1.x backend must be upgraded in lockstep, and the
  long-standard `slf4j-log4j12` should be replaced — Log4j 1.x is
  end-of-life; `slf4j-reload4j` targets reload4j, Ceki's security-maintained
  Log4j 1.x fork created after Log4Shell[^6].
- **Version pinning matters because it is transitive.** Because so many
  libraries depend on `slf4j-api`, the version actually loaded is whatever
  wins dependency mediation. Pin `slf4j-api` explicitly (or via a BOM) so a
  minor upgrade in one library doesn't silently move it.
- **No configuration lives here.** SLF4J has no config file, no log levels,
  no appenders — those all belong to the backend (`logback.xml`,
  `log4j2.xml`, etc.). Debugging "why is nothing at DEBUG showing" is almost
  always a backend-config question, not an SLF4J one.
- **`slf4j-simple` and NOP.** `slf4j-simple` logs to `stderr` and is fine
  for tests and CLIs but has no configuration depth; `slf4j-nop` explicitly
  discards everything and is the correct way to silence logging on purpose.

## When to Use / When Not

**Use when:**
- You are writing a library and must log without dictating a backend to
  consumers — this is the canonical, near-mandatory use case.
- Your application composes many third-party libraries and you want all of
  their logging unified under one configuration.
- You want deferred, parameterized log formatting as a baseline API.

**Avoid / reconsider when:**
- You are writing a self-contained application and are comfortable coding
  directly against Logback or Log4j 2 — the facade buys little and adds one
  more version to manage. (Many still add it for the parameterized API and
  future flexibility.)
- You need structured/JSON logging as a first-class concern — that lives in
  the backend or in newer facades, and SLF4J's core interface is
  string-message-centric (the 2.0 fluent API only partly addresses this).
- You are on a modern greenfield stack and want a facade with built-in
  structured events; evaluate the alternatives below before defaulting.

## Alternatives

- ch.qos.logback/logback — the reference SLF4J backend, by the same author; the usual "just use this" pairing when you control the app.
- apache/logging-log4j2 — full backend and its own facade; use its `log4j-slf4j2-impl` bridge to sit behind SLF4J, or use Log4j2 directly when you want its async loggers and rich config.
- jboss-logging/jboss-logging — facade used across the WildFly/Quarkus ecosystem; prefer it inside that stack.
- Java platform `System.Logger` (JEP 264, JDK 9+) — zero-dependency facade in the JDK itself; use it for libraries that must avoid any external logging dep, accepting a far thinner API.
- google/flogger — fluent logging API from Google; consider when you want a builder-style API and its performance model over SLF4J's.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | ~2005 | First release; static-binding facade by Ceki Gülcü / QOS.ch[^2]. |
| 1.6.0 | 2010 | No-op fallback when no binding is present instead of failing hard. |
| 1.7.0 | 2012 | Long-lived line; varargs parameterized methods, Java 5+. Ran as the de-facto standard for years. |
| 2.0.0 | 2022-08 | ServiceLoader-based `SLF4JServiceProvider`, fluent event-builder API, Java 8+; `StaticLoggerBinder` removed[^4]. |
| 2.0.x | 2022–2026 | Current maintenance line; latest patch releases through 2026[^7]. |

## References

[^1]: SLF4J manual — "Overview of SLF4J." https://www.slf4j.org/manual.html
[^2]: QOS.ch — project sponsor and home of SLF4J, Logback, and reload4j. https://www.qos.ch/
[^3]: SLF4J codes — "Failed to load class org.slf4j.impl.StaticLoggerBinder." https://www.slf4j.org/codes.html#StaticLoggerBinder
[^4]: SLF4J FAQ / news — 2.0.0 adoption of the `ServiceLoader` provider mechanism. https://www.slf4j.org/faq.html#changesInVersion200
[^5]: SLF4J codes — bridging/legacy circular-delegation warning. https://www.slf4j.org/codes.html#unwanted_bridging
[^6]: reload4j — security-maintained fork of Log4j 1.x by QOS.ch. https://reload4j.qos.ch/
[^7]: `org.slf4j` artifacts on Maven Central. https://central.sonatype.com/search?namespace=org.slf4j

## Tags

java, logging, logging-facade, slf4j, jvm, observability, api-abstraction, maven, backend-agnostic, mdc
