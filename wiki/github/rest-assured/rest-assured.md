# rest-assured/rest-assured

> A Java DSL for testing REST services — write HTTP request/response assertions in a given/when/then style without hand-rolling clients or parsers.

[GitHub repo](https://github.com/rest-assured/rest-assured) ·
[License: Apache-2.0](https://github.com/rest-assured/rest-assured/blob/master/LICENSE)

## Overview

REST Assured is a Java library for testing HTTP/REST APIs. Its pitch, stated in the README, is that validating REST services in Java was historically more verbose than in dynamic languages like Ruby or Groovy, and it borrows their ergonomics into the JVM[^1]. First released in 2010, it has become one of the default choices for API integration testing in Java shops, especially alongside JUnit or TestNG.

The core idea is a fluent, BDD-flavored DSL: `given()` sets up the request (params, headers, auth, body), `when()` issues the HTTP verb, and `then()` asserts on the response. Assertions are expressed as GPath expressions over the parsed JSON or XML body, combined with Hamcrest matchers. This lets a test read like a sentence and avoid boilerplate around building requests, serializing bodies, and walking response trees.

The defining tension is that REST Assured optimizes for readability of individual tests at the cost of a large, opinionated dependency footprint and a reliance on global mutable state. It carries Groovy on the classpath (GPath is a Groovy feature), pulls in an HTTP client, JSON/XML binders, and Hamcrest, and exposes much of its configuration through static fields. Convenient for a quick test; a source of classpath conflicts and cross-test pollution at scale.

## Getting Started

Maven (`io.rest-assured` on Maven Central):

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>6.0.1</version>
    <scope>test</scope>
</dependency>
```

A minimal test using static imports:

```java
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@Test
void lottoIdIsFive() {
    given().
        baseUri("https://api.example.com").
    when().
        get("/lotto").
    then().
        statusCode(200).
        body("lotto.lottoId", equalTo(5)).
        body("lotto.winners.winnerId", hasItems(23, 54));
}
```

The `body(...)` path is a GPath expression, not the Jayway JsonPath syntax — a common point of confusion (see Production Notes).

## Architecture / How It Works

REST Assured is a facade over several layers, wired through static configuration:

- **DSL / entry point.** `RestAssured` exposes static methods (`given()`, `when()`, `get()`, etc.) and static mutable configuration (`baseURI`, `port`, `basePath`, `authentication`, `config`, `requestSpecification`). Static imports are the intended usage, which is what makes tests terse.
- **HTTP layer.** Requests are executed via Apache HttpClient under the hood. Timeouts, proxies, SSL, redirects, and connection settings are set through `RestAssuredConfig` / `RestAssured.config()`.
- **Body parsing (GPath).** JSON and XML responses are parsed and queried with GPath, Groovy's path expression language. This is why Groovy is a runtime dependency. The `json-path` and `xml-path` modules can be used standalone to query documents outside a test flow.
- **Assertions.** The `then().body(path, matcher)` form binds a GPath result to a Hamcrest matcher. Failures produce a diff-style message showing the path, expected matcher, and actual value.
- **Object mapping.** Request/response bodies serialize and deserialize through whichever binder is on the classpath — Jackson, Gson, JAXB, or JSON-B (Yasson/Johnzon) — auto-detected. `as(MyDto.class)` maps a response into a POJO.

The library is split into modules so you can pull only what you need: `rest-assured` (the DSL), `json-path`, `xml-path`, `json-schema-validator`, `spring-mock-mvc` and `spring-web-test-client` (drive Spring controllers without a running server), `kotlin-extensions`, and `scala-support`. The Spring modules are the notable integration surface — they let you exercise the same DSL against `MockMvc` rather than a live socket.

## Production Notes

- **Global mutable state is the primary footgun.** `RestAssured.baseURI`, `port`, `authentication`, and friends are static fields. Setting them in one test leaks into others unless you call `RestAssured.reset()` in teardown. This also makes the static API unsafe for parallel test execution — tests running concurrently share and clobber the same config. The isolation-safe pattern is to build a `RequestSpecification` (via `RequestSpecBuilder`) per test/thread and use it explicitly instead of the static defaults.
- **Groovy on the classpath.** GPath means Groovy is a transitive runtime dependency. This bloats the dependency tree and is a recurring source of version conflicts, particularly in projects that already use Groovy (Gradle build logic, or Spock, which is itself Groovy-based). Mismatched Groovy versions surface as obscure `NoSuchMethodError` / `MethodMissing` failures at runtime, not compile time.
- **GPath ≠ Jayway JsonPath.** The `body("a.b[0]")` syntax is GPath, not the `$.a.b[0]` JsonPath used by `com.jayway.jsonpath`. Expressions copied from JsonPath tutorials will not work, and the two produce different results for filters and wildcards. Know which one a given snippet targets.
- **Java baseline moves with major versions.** 6.0.0 (2025-12) raised the minimum to Java 17+ and moved to Groovy 5, with Spring 7 and Jackson 3 support[^2]. Upgrading a major version can force a JDK and framework bump at the same time; the 5.5.x line was kept alive (5.5.7, 2026-01) to backport Spring 7 MockMvc support for teams that could not jump to 6.x.
- **Spring version bleed.** Historically the Spring modules pulled their own Spring version onto the classpath, requiring manual exclusions on Spring Boot. 6.0.1 (2026-07) explicitly fixed this so the Spring modules target Spring 7 without dragging a conflicting version in[^3].
- **It is a test-time library.** Meant for `test` scope. It is a client for asserting against APIs, not a production HTTP client or a load-testing tool — response-time assertions exist but are coarse.

## When to Use / When Not

**Use when:**
- You write integration/API tests in Java (or Kotlin) against JSON/XML REST services and want readable given/when/then assertions.
- You're on the Spring stack and want to drive controllers via MockMvc or WebTestClient with the same DSL.
- You need quick, expressive body assertions without hand-writing a client and a parser per test.

**Avoid when:**
- You need parallel test execution and want to avoid the static-state hazards — you can, but only by disciplined use of explicit specifications.
- You want a minimal dependency footprint; the Groovy + HTTP client + binder stack is heavy for a small project.
- You're testing gRPC, GraphQL, or WebSocket APIs — it's built around request/response HTTP verbs.
- Your team is on a language with a native idiomatic client (e.g., Kotlin projects may prefer Ktor's test client, or plain `HttpClient` + a JSON assertion library).

## Alternatives

- karatelabs/karate — full API test framework with its own Gherkin-style language; use it when you want tests independent of Java and built-in parallelism/reporting.
- Apache HttpComponents (httpclient) + a JSON assertion library — use when you want a minimal footprint and full control over the client.
- assertj/assertj-core with a plain HTTP client — use when you prefer AssertJ's fluent assertions and want to avoid Groovy/global state.
- wiremock/wiremock — complementary, not a replacement: use it to stub/mock the services REST Assured tests against.
- intuit/karate (see karatelabs/karate) or Postman/Newman — use when non-Java stakeholders need to read or run the tests.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2010-10 | First release; given/when/then DSL over GPath[^1]. |
| 4.0 | 2019 | Dependency and HTTP layer modernization. |
| 5.0 | 2022 | Java baseline and dependency updates. |
| 5.5.7 | 2026-01-16 | Backported Spring 7 MockMvc support onto the 5.x line[^2]. |
| 6.0.0 | 2025-12-12 | Java 17+, Groovy 5, Spring 7 + Jackson 3 support[^2]. |
| 6.0.1 | 2026-07-10 | Spring modules target Spring 7 without pulling their own version; rounds out Jackson 3[^3]. |

## References

[^1]: REST Assured README. https://github.com/rest-assured/rest-assured/blob/master/README.md
[^2]: REST Assured Release Notes 6.0 and change log. https://github.com/rest-assured/rest-assured/wiki/ReleaseNotes60
[^3]: REST Assured change log (6.0.1, 2026-07-10). https://raw.githubusercontent.com/rest-assured/rest-assured/master/changelog.txt

## Tags

java, kotlin, rest, api-testing, integration-testing, http, json, xml, test-automation, hamcrest, groovy, bdd
