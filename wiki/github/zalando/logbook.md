# zalando/logbook

> An extensible Java library that captures complete HTTP request/response pairs — including bodies — for audit, debugging, and traffic analysis.

[GitHub repo](https://github.com/zalando/logbook) ·
[License: MIT](https://github.com/zalando/logbook/blob/main/LICENSE)

## Overview

Logbook is a JVM library, open-sourced by Zalando, for logging full HTTP messages — request line, headers, and body — on both the client and server side of an application[^1]. It exists to answer a question that framework access logs cannot: "what exactly did we receive from / send to this caller, byte for byte?" That makes it a tool for audit trails, incident forensics, and API debugging rather than for metrics or tracing. As of 2026 it sits around 2,050 stars with steady maintenance and a wide catalogue of framework adapters.

The defining tension is that logging bodies is expensive and dangerous. Bodies must be buffered so the application can still consume them, which costs memory; and bodies routinely contain secrets and PII, which must be scrubbed before they hit a log file. Logbook's entire design — a pipeline of conditions, filters, formatters, and writers — is organized around making that expense opt-out (skip health checks, drop binary streams) and making redaction opt-in-by-default (Authorization headers and known token fields are filtered out of the box). It is not a general logging framework; it feeds SLF4J/Logback and expects you to already have one.

The library is modular to a fault: `logbook-core` plus a separate adapter artifact per integration (Servlet, Apache HttpClient 4/5, OkHttp 2/3, JAX-RS, Netty, Ktor, Spring Boot starter). You pull only the adapters your stack uses; they all share one version number and one `Logbook` instance.

## Getting Started

Maven, using the Spring Boot starter (auto-configures a servlet filter):

```xml
<dependency>
    <groupId>org.zalando</groupId>
    <artifactId>logbook-spring-boot-starter</artifactId>
    <version>${logbook.version}</version>
</dependency>
```

Logbook writes at `TRACE`, so nothing appears until you raise the level — the single most common "it's not working" cause:

```properties
logging.level.org.zalando.logbook: TRACE
```

Without Spring Boot, build an instance and wire it manually:

```java
Logbook logbook = Logbook.builder()
    .condition(exclude(requestTo("/health"), requestTo("/admin/**")))
    .headerFilter(authorization())        // redacts Authorization by default anyway
    .queryFilter(accessToken())
    .build();

// e.g. as a servlet filter
filterChain.addFilter(new LogbookFilter(logbook));
```

## Architecture / How It Works

Every message flows through four phases, each an interface with a sensible default[^2]:

1. **Conditional** — a `Predicate<HttpRequest>` decides whether the pair is logged at all. This is where you exclude load-balancer health checks, management endpoints, and binary content-types before paying any cost.
2. **Filtering** — redaction. Six filter types operate at different granularities: `QueryFilter`, `PathFilter`, `HeaderFilter`, `BodyFilter` (high-level, cover ~90% of needs), and `RequestFilter` / `ResponseFilter` (low-level escape hatches). Filters chain and run consecutively.
3. **Formatting** — turn the message into a string. Two built-ins: `DefaultHttpLogFormatter` (raw HTTP-looking text, meant for local dev) and `JsonHttpLogFormatter` (one JSON object per message, meant for log aggregation).
4. **Writing** — where the string goes. The default writer emits to SLF4J; you can point it at a stream or a custom sink.

Two structural facts matter most. First, **body buffering**: to log a body while still letting the application read it, the adapter wraps the request/response and tees the byte stream. Binary, multipart, and streaming bodies are replaced with placeholders by default precisely because buffering them is unbounded. Second, the **Strategy** abstraction (added in 2.0) sits above the phases and controls the coarse decision of what to log and when — for example `BodyOnlyIfStatusAtLeastStrategy` logs bodies only for error responses, avoiding the cost on the happy path[^3].

A later addition, the **Attribute Extractor** (3.4.0), lets you pull structured key/value pairs out of a message — the motivating case was extracting a `sub` claim from a JWT in the Authorization header — and attach them to the log entry as an `attributes` object[^4]. JSON body redaction can also be done structurally via an experimental JSONPath filter (`jsonPath("$.password").delete()`).

JSON handling is decoupled from a specific Jackson major version: Logbook detects Jackson 2 or Jackson 3 on the classpath and adapts, preferring 3 if both are present, and disabling JSON formatting (not core logging) if neither is.

## Production Notes

- **TRACE-level gate.** Logs are emitted under logger `org.zalando.logbook` at TRACE. Forget to raise it and you get silence with no error. Raise it too broadly and you drown in traffic. Scope the level to the specific adapter package if needed.
- **Body buffering is a memory cost, not free.** Every logged body is held in memory to be teed back to the application. Large uploads/downloads, file transfers, and SSE/streaming endpoints should be excluded by condition or left as placeholder replacements — otherwise you risk heap pressure and, for streams, breaking the response semantics.
- **Redaction is your responsibility beyond the defaults.** Out of the box it strips `Authorization`, `access_token`/`refresh_token` (JSON), and `password`/`client_secret` (form). Any other secret or PII field in a body — emails, tokens with non-standard names, national IDs — will be logged verbatim unless you add a `BodyFilter`. Audit your payloads before enabling body logging in production.
- **Servlet wrapping interactions.** The servlet adapter wraps request/response objects; on rare occasions this interferes with other filters that also wrap, with async/streaming responses, or with frameworks that read the input stream in unusual ways. Filter ordering matters.
- **It is not tracing.** Logbook has a correlation id to pair a request with its response, but it produces human/aggregator-readable message dumps, not spans. For distributed context propagation and latency analysis you want an observability stack instead of, or alongside, Logbook.
- **Version-baseline churn.** The current line requires Java 17 and targets Spring 7 / Spring Boot 4, JAX-RS 3.x (Jakarta namespace), and Jackson 2 or 3. Upgrading from older Logbook on Spring Boot 2 / `javax.*` is a coordinated jump, not a drop-in bump.

## When to Use / When Not

**Use when:**
- You need auditable, full request/response capture (bodies included) for compliance or incident forensics.
- You have a mixed stack (servlet inbound + several HTTP clients outbound) and want one consistent, filterable log format across all of it.
- You need fine-grained control over what gets logged and redacted, with JSON output for aggregation.

**Avoid when:**
- You only need latency, error rates, and request counts — reach for tracing/metrics instead.
- Your traffic is dominated by large binary/streaming payloads where body buffering is a non-starter.
- You want zero extra dependencies and only need request-line/URI logging — a built-in framework filter is enough.

## Alternatives

- open-telemetry/opentelemetry-java-instrumentation — use instead when you want spans, metrics, and context propagation rather than raw request/response payload dumps.
- square/okhttp — its bundled `logging-interceptor` covers the case when your entire surface is a single OkHttp client and you want a one-liner, not a pipeline.
- spring-projects/spring-framework — the built-in `CommonsRequestLoggingFilter` handles inbound request URI/params logging with no extra dependency, but does not log response bodies.
- micrometer-metrics/micrometer — use when the goal is observability signals (timers, distribution summaries) exported to a backend, not message logging.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | Initial open-source release of Zalando's internal HTTP logging tooling[^1]. |
| 2.0 | ~2018 | Strategy pattern introduced; flexible/partial logging beyond the original rigid pairing[^3]. |
| 3.x | ~2021 | Module reorganization; Jakarta / newer Java and Spring Boot baselines. |
| 3.4.0 | ~2023 | Attribute Extractor added (e.g. JWT claim extraction)[^4]. |
| current | 2026 | Java 17 baseline, Spring 7 / Spring Boot 4, JAX-RS 3.x, Jackson 2 & 3 auto-detection. |

## References

[^1]: zalando/logbook — repository and README. https://github.com/zalando/logbook
[^2]: Logbook README, "Phases" (Conditional / Filtering / Formatting / Writing). https://github.com/zalando/logbook#phases
[^3]: Logbook README, "Strategy" and `Strategy` interface (introduced in 2.0). https://github.com/zalando/logbook#strategy
[^4]: Logbook README, "Attribute Extractor" (since 3.4.0, per issue #381). https://github.com/zalando/logbook/issues/381

## Tags

java, jvm, http-logging, observability, request-response, servlet, spring-boot, audit, jax-rs, okhttp, redaction
