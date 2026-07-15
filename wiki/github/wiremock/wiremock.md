# wiremock/wiremock

> An HTTP mock server for simulating APIs you depend on — as a JUnit rule, a standalone process, or a container.

GitHub repo: https://github.com/wiremock/wiremock ·
Official website: https://wiremock.org ·
License: Apache-2.0

## Overview

WireMock is a tool for standing in for the HTTP services your code talks to. You describe request→response contracts ("stubs"), point your application at WireMock instead of the real dependency, and get a deterministic backend that returns exactly what your test expects — including faults, latency, and stateful sequences that a real service is awkward to coerce into producing. It started in 2011 as a Java library written by Tom Akehurst[^1] and has since grown into a standalone server, a Docker image, a Testcontainers module, and a set of client wrappers/ports across other language ecosystems.

The project's center of gravity is still the JVM. The core is Java, the fluent DSL is Java-first, and the most polished integrations (JUnit 4 `Rule`, JUnit 5 extension, Spring Boot) are Java. Everything else — the REST admin API, JSON stub files, Docker, the various language ports — is built around that core. If you are not on the JVM you interact with WireMock over its HTTP admin API or through a community wrapper rather than the native DSL.

WireMock's defining tension is scope. It is deliberately protocol-focused (HTTP request matching + response generation) and does not try to validate against an API schema by default — matching is what *you* write, not what an OpenAPI document dictates. That keeps it flexible and unopinionated, but means a WireMock stub can happily drift out of sync with the real API it imitates. The commercial WireMock Cloud[^2] (from the same company, formerly MockLab) sells the schema-aware, OpenAPI-driven, multi-tenant layer that open-source WireMock intentionally leaves out.

## Getting Started

Maven (test scope):

```xml
<dependency>
  <groupId>org.wiremock</groupId>
  <artifactId>wiremock</artifactId>
  <version>3.x.x</version>
  <scope>test</scope>
</dependency>
```

JUnit 5 test with the WireMock extension:

```java
@WireMockTest
class ApiClientTest {
  @Test
  void returnsStubbedUsers(WireMockRuntimeInfo wm) {
    stubFor(get("/api/users")
        .willReturn(okJson("[{\"id\":1,\"name\":\"Ada\"}]")));

    var body = new ApiClient(wm.getHttpBaseUrl()).fetchUsers();

    assertThat(body).contains("Ada");
    verify(getRequestedFor(urlEqualTo("/api/users")));
  }
}
```

Standalone, for use from any language or CI step:

```bash
java -jar wiremock-standalone-3.x.x.jar --port 8080
# or
docker run -it --rm -p 8080:8080 wiremock/wiremock
# then POST stub definitions to http://localhost:8080/__admin/mappings
```

## Architecture / How It Works

At runtime WireMock is an embedded HTTP server (Jetty) plus a matching engine and a stub store. An incoming request is scored against every registered stub mapping; the best match wins and its `ResponseDefinition` is rendered and returned. Mappings can be registered three ways that all funnel into the same store: the Java DSL (`stubFor(...)`), JSON files under `mappings/` loaded at startup, or JSON `POST`ed to the `/__admin` REST API at runtime. The admin API is the real substrate — the Java DSL is a typed client over it.

The matching system is the core value. Any facet of a request — method, URL (equality, regex, path templates), headers, query params, cookies, and body — can carry a matcher, and body matchers include JSONPath, XPath, JSON/XML equality (order-insensitive), and regex. Matchers compose, and unmatched requests return a diff-style "closest stub" report that is genuinely useful for debugging why a stub didn't fire.

Responses can be static or generated. Response templating uses Handlebars[^3], so a stub can echo request values back (`{{request.query.id}}`), inject dates, or shape output from the incoming request — which is how WireMock simulates dynamic APIs without code. Stateful behavior is modeled with **Scenarios**: named state machines where a stub only matches in a given state and transitions on match, letting you script "first call returns 202, poll returns 200" flows.

Two other pillars: **record/playback**, where WireMock proxies to a real backend and captures the traffic as reusable stubs; and **extensions** — request filters, response transformers, and admin-API extensions loaded on the classpath, which is how gRPC, GraphQL, and webhook support are delivered as add-ons rather than core.

The 2.x→3.x boundary is the most consequential internal change. WireMock 3 moved to the Jakarta EE namespace (Jetty 11, `jakarta.servlet` instead of `javax.servlet`), raised the minimum to Java 11, changed the Maven coordinates to `org.wiremock`, and reworked the extension API[^4]. Code written against 2.x extension interfaces does not port unchanged.

## Production Notes

- **It is a test/dev tool, not a production gateway.** People do run WireMock standalone as a long-lived mock in shared environments, and it holds up, but there is no built-in auth, multi-tenancy, or persistence beyond the mappings on disk. Treat a shared instance as mutable global state — one test's `POST /__admin/reset` wipes another's stubs.
- **Coordinate change 2.x→3.x is a real migration, not a version bump.** The `javax`→`jakarta` move breaks any custom extension and any host application still on Jetty 9 / older servlet APIs. Teams pinned to Java 8 cannot move to 3.x at all. The groupId change (`com.github.tomakehurst` → `org.wiremock`) also means your dependency management won't auto-upgrade across the boundary.
- **Classpath conflicts are the classic footgun.** WireMock shades many dependencies in the standalone JAR, but the thin `wiremock` artifact can collide with your app's Jetty, Jackson, Guava, or Handlebars versions. When stubs behave strangely or startup fails with `NoSuchMethodError`, the standalone/shaded JAR usually resolves it.
- **Random vs fixed ports.** Binding a fixed port (8080) makes tests flaky under parallel execution; use dynamic ports (`.dynamicPort()`) and read the assigned port back via `WireMockRuntimeInfo`. Leaking servers across tests (forgetting to stop) exhausts ports in large suites.
- **Response templating is powerful and unsandboxed by intent.** Enabling global templating means any stub body is evaluated as a Handlebars template; a literal `{{` in fixture data will be interpreted unless escaped. Enable templating per-stub when you don't need it everywhere.
- **HTTPS and HTTP/2** work but need explicit configuration (keystore, `.httpsPort()`); the default is plain HTTP. Proxy/record mode against TLS backends requires trusting or bypassing certs.

## When to Use / When Not

**Use when:**
- You need deterministic HTTP dependencies in tests — flaky third parties, rate-limited sandboxes, or a backend that doesn't exist yet.
- You want to simulate faults, latency, and stateful call sequences that real services won't produce on demand.
- You're on the JVM and want a mature, well-integrated (JUnit/Spring/Testcontainers) mocking layer.
- You need a language-agnostic mock server that any client can drive over a REST admin API.

**Avoid when:**
- You want contract testing that a mock stays faithful to the provider's schema — reach for consumer-driven contract tools (Pact) or an OpenAPI mock; WireMock won't tell you your stub drifted.
- You're mocking browser `fetch`/XHR in a frontend test — an in-process interceptor (MSW) is lighter than a real server.
- You need multi-protocol simulation (TCP, SMTP, arbitrary binary) — WireMock is HTTP-first.
- You want managed, schema-aware, team-shared mocks without operating a server — that is the commercial WireMock Cloud pitch, not the OSS core.

## Alternatives

- mock-server/mockserver — closest JVM peer; HTTP mock/proxy with expectations. Use instead when you want proxying/verification-heavy workflows or a different DSL.
- stoplightio/prism — mocks *from* an OpenAPI/Swagger document. Use when schema fidelity matters more than hand-written matchers.
- mswjs/msw — intercepts requests in-process (browser + Node). Use for frontend/JS testing where a real server is overkill.
- bbyars/mountebank — multi-protocol (HTTP, TCP, SMTP) over the wire. Use when you must mock beyond HTTP.
- spectolabs/hoverfly — proxy-based API simulation with capture/replay. Use when a middle-man proxy fits your topology better than a target server.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial library | 2011 | Created by Tom Akehurst as a Java HTTP mocking library[^1]. |
| 2.0 | ~2016–2017 | Long-lived 2.x line: JUnit rule, standalone JAR, record/playback, templating, Jetty 9 / `javax.servlet`, Java 8. |
| 3.0.0 | 2023-07 | Jakarta namespace (Jetty 11), Java 11+, `org.wiremock` coordinates, reworked extension API[^4]. |
| 3.x | 2023– | Active 3.x line; gRPC, GraphQL, webhooks, Testcontainers module as extensions/modules. |

## References

[^1]: WireMock README and project history — "Started in 2011 as a Java library by Tom Akehurst." https://github.com/wiremock/wiremock
[^2]: WireMock Cloud — commercial hosted offering from WireMock Inc. (formerly MockLab). https://www.wiremock.io/
[^3]: Response templating uses the Handlebars templating engine. https://wiremock.org/docs/response-templating/
[^4]: WireMock 3.x documentation, including the 2.x→3.x migration and Jakarta/Jetty changes. https://wiremock.org/docs/

## Tags

java, http, api-mocking, mock-server, testing, stubbing, jvm, integration-testing, service-virtualization, junit
