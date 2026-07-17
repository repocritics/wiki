# graphql-java-kickstart/graphql-spring-boot

> Spring Boot auto-configuration starters that wired graphql-java into a Boot app in one dependency — now archived in favor of Spring's official GraphQL support.

[GitHub repo](https://github.com/graphql-java-kickstart/graphql-spring-boot) ·
[Official website](https://www.graphql-java-kickstart.com/spring-boot/) ·
[License: MIT](https://github.com/graphql-java-kickstart/graphql-spring-boot/blob/master/LICENSE.md)

## Overview

This project is a set of Spring Boot starters that auto-configure a GraphQL
server from the graphql-java stack. Adding `graphql-spring-boot-starter` to a
Boot web application exposes a `/graphql` servlet as soon as a `GraphQLSchema`
bean (or a supported schema library) is on the classpath, and companion starters
drop in browser IDEs — GraphiQL, Altair, GraphQL Playground, and Voyager — behind
their own endpoints. It was, for most of the 2018–2022 period, the default way to
run GraphQL on Spring Boot before an official option existed.

The repository is a hard fork of `oembedler/graphql-spring-boot`, taken over by
the graphql-java-kickstart organization "due to inactivity" of the original[^1].
It sits on top of the same org's `graphql-java-servlet` and `graphql-java-tools`
libraries, which in turn wrap `graphql-java` (the reference GraphQL execution
engine for the JVM). The starters are glue: property binding, bean discovery, and
endpoint registration. The GraphQL semantics live one layer down.

As of December 2022 the project is **archived and unmaintained**[^2]. The
maintainers state that because Spring now ships first-party GraphQL support, new
projects should use Spring for GraphQL (`spring-graphql`) instead, and that
teams stuck on this library should fork it themselves. The final published
coordinate is `14.0.0`, targeting Java 8 and Spring Boot 2.x[^3]. This is
important context for anyone evaluating it today: it is a working, once-popular
library with no upstream, no Spring Boot 3 / Jakarta namespace migration, and no
security backports.

## Getting Started

Maven — pull in the starter (a matching `graphql-spring-boot-starter-test`
exists for test scope):

```xml
<dependency>
  <groupId>com.graphql-java-kickstart</groupId>
  <artifactId>graphql-spring-boot-starter</artifactId>
  <version>14.0.0</version>
</dependency>
```

With `graphql-java-tools` on the classpath, put a `*.graphqls` schema under
`src/main/resources` and back it with resolver beans:

```graphql
# src/main/resources/schema.graphqls
type Query { hello(name: String): String }
```

```java
@Component
public class Query implements GraphQLQueryResolver {
    public String hello(String name) {
        return "Hello, " + (name == null ? "world" : name);
    }
}
```

The server auto-registers at `/graphql`; set `graphql.graphiql.enabled=true` to
serve the GraphiQL IDE at `/graphiql`.

## Architecture / How It Works

The stack is layered and the starter is the thinnest layer:

- **graphql-java** — the execution engine: parsing, validation, and field
  resolution against a `GraphQLSchema`.
- **graphql-java-tools** — the schema-first bridge. It reads `*.graphqls` SDL
  files and wires them to `GraphQLResolver` beans by convention (`GraphQLQueryResolver`,
  `GraphQLMutationResolver`, `GraphQLSubscriptionResolver`). Alternatively,
  `graphql-java-annotations` (Enigmatis) can be selected via
  `graphql.schema-strategy=annotations` for a code-first build.
- **graphql-java-servlet** — a plain `HttpServlet` that speaks the GraphQL-over-HTTP
  conventions (GET/POST, batching, multipart file upload, WebSocket subscriptions).
- **graphql-spring-boot** (this repo) — auto-configuration classes that discover
  those beans, bind `graphql.servlet.*` / `graphql.tools.*` properties, mount the
  servlet, and register the IDE endpoints.

Because the transport is servlet-based, the runtime model is thread-per-request
and blocking. There is a `PER_REQUEST_WITH_INSTRUMENTATION` context setting and
an async mode, but this is not a reactive (WebFlux) stack — that is the sharpest
architectural difference from Spring for GraphQL, which is built on the reactive
`GraphQlSource` abstraction and works with both MVC and WebFlux. Subscriptions
here ride graphql-java-servlet's WebSocket implementation rather than Reactor
`Publisher` streams.

The IDE starters (GraphiQL, Altair, Playground, Voyager) are independent: each is
a small controller that serves a bundled static build of the respective tool,
with a CDN toggle and a large surface of pass-through configuration properties.
They are convenience only and have no bearing on schema execution.

## Production Notes

- **It is archived — treat it as end-of-life.** No fixes land upstream. Any CVE in
  a transitive dependency (graphql-java, Jackson, the bundled IDE JS assets) is
  your problem to patch by overriding versions or forking[^2].
- **Spring Boot 3 / Jakarta EE is unsupported.** The library targets `javax.servlet`
  and Spring Boot 2.x. Moving to Boot 3 (Jakarta namespace) is effectively a
  migration off this library, not an upgrade of it. This is the single biggest
  practical blocker in 2026.
- **Schema-first coupling to graphql-java-tools.** The happy path assumes SDL
  `*.graphqls` files plus marker-interface resolver beans. Field resolution binds
  by method name and signature; mismatches surface as startup or first-query
  errors rather than compile errors. The README also flags a known
  `NoClassDefFoundError` class of failures when mixing incompatible
  graphql-java-tools versions[^3].
- **IDE endpoints and introspection default to enabled.** GraphiQL/Altair/Playground/
  Voyager and the introspection query are convenient in dev but should be disabled
  or access-controlled in production — an open graphical explorer is a common
  information-exposure footgun. CORS is likewise on by default for `/graphql/**`;
  override `graphql.servlet.cors` rather than trusting the default origins.
- **Migration target is Spring for GraphQL.** The move is non-mechanical: SDL files
  largely carry over, but resolver wiring changes from marker interfaces to
  `@SchemaMapping`/`@QueryMapping` controllers, and the servlet/property model is
  replaced by `spring-boot-starter-graphql`.

## When to Use / When Not

**Use when:**
- You maintain an existing Spring Boot 2.x service already built on these starters
  and need to keep it running without a rewrite.
- You need a specific bundled IDE (Altair, Voyager) wired with minimal config on a
  legacy stack.

**Avoid when:**
- You are starting a new project — use Spring for GraphQL instead.
- You are on (or moving to) Spring Boot 3 / Jakarta EE, or a reactive WebFlux stack.
- You require an actively maintained dependency with security backports.

## Alternatives

- spring-projects/spring-graphql — the official successor; first-party Spring Boot
  3 support, MVC and WebFlux, annotation-based controllers. The recommended target.
- graphql-java/graphql-java — use directly when you want the engine without any
  Spring auto-configuration and are willing to wire transport yourself.
- Netflix/dgs-framework — Netflix's annotation-driven, code-first GraphQL framework
  for Spring Boot; a maintained alternative that later converged with spring-graphql.
- expediagroup/graphql-kotlin — use when your service is Kotlin-first and you want
  reflection-based code-first schema generation.
- hasura/graphql-engine — use when you want an auto-generated GraphQL API over a
  database rather than hand-written JVM resolvers.

## History

| Version | Date | Notes |
|---------|------|-------|
| fork | 2017-03-18 | Repo created as a fork of oembedler/graphql-spring-boot[^1]. |
| 5.x | 2019 | Widely-adopted era; graphql-java-tools schema-first as the norm. |
| 14.0.0 | 2022 | Final release line; Java 8, Spring Boot 2.x[^3]. |
| archived | 2022-12 | Marked unmaintained; users directed to Spring for GraphQL[^2]. |

## References

[^1]: Repository description, "GraphQL and GraphiQL Spring Framework Boot Starters — Forked from oembedler/graphql-spring-boot due to inactivity." https://github.com/graphql-java-kickstart/graphql-spring-boot
[^2]: Archive notice, README: "THIS REPOSITORY HAS BEEN ARCHIVED AND IS NO LONGER BEING MAINTAINED … We encourage you to start using Spring for GraphQL instead." https://spring.io/projects/spring-graphql
[^3]: README — Requirements and Downloads (Java 1.8, Spring Boot 2.x, coordinate `graphql-spring-boot-starter:14.0.0`) and the graphql-java-tools FAQ. https://www.graphql-java-kickstart.com/spring-boot/

## Tags

java, spring-boot, graphql, api, backend, starter, graphql-java, archived, server, schema-first
