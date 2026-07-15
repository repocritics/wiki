# OpenFeign/feign

> A declarative HTTP client for Java: annotate an interface, get a working REST client at runtime via a dynamic proxy.

[GitHub repo](https://github.com/OpenFeign/feign) ·
[License: Apache-2.0](https://github.com/OpenFeign/feign/blob/master/LICENSE)

## Overview

Feign turns a Java interface into an HTTP client. You declare methods with annotations (`@RequestLine`, `@Param`, `@Headers`), and at build time Feign generates a JDK dynamic proxy that translates each method call into an HTTP request, encodes the arguments, executes the call, and decodes the response back into your return type. It was inspired by Square's Retrofit and by JAX-RS 2.0, and originated inside Netflix as the client-binding layer for the Denominator project before being spun out into the independent OpenFeign organization around 2016[^1].

The defining tradeoff is declarative simplicity in exchange for a synchronous, blocking execution model. Feign is deliberately small: the core has almost no dependencies and delegates everything pluggable — the transport (`Client`), serialization (`Encoder`/`Decoder`), retries (`Retryer`), error mapping (`ErrorDecoder`), and logging — to interfaces you supply. That minimalism is why it composes cleanly with almost any JSON/XML library and HTTP stack, but it also means Feign does little on its own; a bare `feign-core` client with no decoder cannot even parse a JSON body.

In practice most Java teams do not use this library directly. They consume it through Spring Cloud OpenFeign (`@FeignClient`), which swaps Feign's native annotation contract for `SpringMvcContract` so that Spring MVC annotations (`@GetMapping`, `@RequestParam`) work instead[^2]. That layering matters for anyone debugging Feign: the annotations, contract, and configuration you see in a Spring Boot app are Spring Cloud's, not the ones documented in this repo.

## Getting Started

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-core</artifactId>
    <version>13.5</version>
</dependency>
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-gson</artifactId>
    <version>13.5</version>
</dependency>
```

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  class Contributor {
    String login;
    int contributions;
  }
}

GitHub github = Feign.builder()
    .decoder(new GsonDecoder())
    .target(GitHub.class, "https://api.github.com");

List<Contributor> cs = github.contributors("OpenFeign", "feign");
```

`Feign.builder()` is the single entry point; every customization (client, encoder, decoder, interceptors, retryer, timeouts) is chained here before `.target()` returns the proxy.

## Architecture / How It Works

Feign's core is a small pipeline wrapped around a JDK `Proxy`:

1. **Contract** — parses the interface's annotations into a list of `MethodMetadata` (request template, parameter indexes, expected return type). The default `Contract.Default` reads Feign's own annotations; `SpringMvcContract` (Spring Cloud) and `JAXRSContract` (feign-jaxrs) are drop-in replacements that read a different annotation vocabulary against the same metadata model.
2. **RequestTemplate** — an annotation becomes a template with RFC 6570 Level 1 URI expressions. At call time `@Param` values are expanded (and pct-encoded) into the template to produce a concrete `Request`.
3. **Encoder** — turns method arguments (typically a POJO body) into the request body. Left unset, only bodies that are already `String`/`byte[]` work.
4. **Client** — executes the `Request` and returns a `Response`. The default is `Client.Default`, backed by `java.net.HttpURLConnection`. Alternatives are separate modules: `feign-okhttp`, `feign-hc5` (Apache HttpClient 5), `feign-java11` (the JDK `HttpClient`), `feign-googlehttpclient`.
5. **Decoder** — converts the `Response` body into the declared return type. `ErrorDecoder` handles non-2xx responses, mapping them to exceptions; `Retryer` decides whether to replay.
6. **RequestInterceptor**s run on every request (the standard place to inject auth headers). `Logger` observes the exchange at a configurable `Logger.Level`.

The whole model is synchronous and one-request-per-call. `AsyncFeign` exists in the core for `CompletableFuture`-returning interfaces, and community reactive wrappers exist, but the mainline design is blocking I/O. Circuit breaking is a separate module (`feign-hystrix`, now effectively dead, superseded by `feign-micrometer` for metrics and Spring Cloud CircuitBreaker/Resilience4j integrations at the framework layer).

Because everything is an interface, Feign is trivial to unit test — you can assert on `RequestTemplate` output without a network — which is one of the project's stated original design goals[^3].

## Production Notes

- **You are probably using Spring Cloud OpenFeign, not this.** Almost all production Feign usage is via `@FeignClient`. Behavior around contracts, default encoders/decoders (Jackson vs Gson), timeouts, and retry defaults is set by Spring Cloud and Spring Boot autoconfiguration, and its release train is versioned independently of this repo. Read Spring Cloud's docs for those defaults, not the README here[^2].
- **The default client is `HttpURLConnection`.** It has no connection pooling and limited control over timeouts/redirects. Any throughput-sensitive service should switch to `feign-hc5` or `feign-okhttp` explicitly; this is the single most common performance footgun.
- **Retryer defaults retry.** `Retryer.Default` retries on `IOException` and on retryable `ErrorDecoder` results, which can silently amplify load or duplicate non-idempotent POSTs during backend brownouts. Set `Retryer.NEVER_RETRY` unless you have deliberately made the endpoint idempotent.
- **No encoder means no body serialization.** A `feign-core`-only build gives cryptic failures on the first POJO body. Always add a codec module (`feign-jackson`, `feign-gson`).
- **URL encoding surprises.** By default `@RequestLine` does not encode slashes, and `+` in query strings is encoded to `%2B` (not treated as space). Systems that expect legacy `+`-as-space semantics will see mismatches. `decodeSlash` and explicit encoding flags exist but must be set per parameter[^3].
- **Long-aspirational roadmap.** Async, reactive, and response caching have sat in the README's "short/medium term" list across several major versions; treat them as partially-available extras, not core guarantees, and pin versions.
- **Java baseline.** Feign 10.x+ targets Java 8; the 9.x line was the last to support JDK 6. Recent releases track newer JDKs but check the module you add (e.g. `feign-java11`) for its own baseline[^3].

## When to Use / When Not

**Use when:**
- You call REST/SOAP APIs from the JVM and want interface-driven clients with minimal boilerplate.
- You already run Spring Cloud and want `@FeignClient` with load-balancing and circuit-breaker integration.
- You want to keep serialization and transport pluggable and easy to mock in tests.

**Avoid when:**
- You need non-blocking/reactive I/O at scale — Spring `WebClient` or a native async client fits better than bolting async onto Feign.
- You're on Spring Boot 3 / Spring 6 without the rest of Spring Cloud — the built-in `@HttpExchange` HTTP interface client covers the common case with no extra dependency.
- You need streaming, HTTP/2 multiplexing, or fine-grained connection control as first-class concerns.
- You want a single library that ships batteries-included; Feign requires you to assemble the codec and client yourself.

## Alternatives

- square/retrofit — the original declarative-client inspiration; leaner, OkHttp-native, strong on Android. Use instead when you want a single well-integrated client stack rather than pluggable pieces.
- spring-projects/spring-framework — Spring 6's `@HttpExchange` HTTP interface clients. Use instead when you're already on Spring Boot 3 and don't need Spring Cloud's extras.
- OpenAPITools/openapi-generator — generates typed clients from an OpenAPI spec. Use instead when the API has a spec and you'd rather generate than hand-declare.
- micronaut-projects/micronaut-core — compile-time declarative HTTP client with no reflection. Use instead when you want AOT/GraalVM-friendly clients without runtime proxies.
- square/okhttp — the imperative baseline. Use instead when you want explicit request/response control and no annotation layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013 | Created at Netflix as the client binder for Denominator[^1]. |
| — | 2016 | Moved out of Netflix into the independent OpenFeign org[^1]. |
| 9.x | 2017–2018 | Last line with JDK 6 support. |
| 10.0 | 2018 | Java 8 baseline; group id `io.github.openfeign`[^3]. |
| 11.x | 2021 | Roadmap targets async/reactive/caching (largely still in progress). |
| 12.x | 2022 | Continued module and JDK-compat maintenance. |
| 13.x | 2023–2026 | Current line; actively maintained (last push 2026-07-14). |

## References

[^1]: Feign origin and move to OpenFeign — project README history and Netflix/Denominator lineage. https://github.com/OpenFeign/feign
[^2]: Spring Cloud OpenFeign — `@FeignClient` and `SpringMvcContract`. https://docs.spring.io/spring-cloud-openfeign/reference/
[^3]: Feign README — usage, contract, templates, and Java version compatibility. https://github.com/OpenFeign/feign/blob/master/README.md

## Tags

java, http-client, declarative-client, rest-client, jvm, spring-cloud, annotations, dynamic-proxy, jax-rs, api-client
