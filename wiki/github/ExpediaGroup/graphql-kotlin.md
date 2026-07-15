# ExpediaGroup/graphql-kotlin

> A code-first GraphQL toolkit for Kotlin — generate your schema from Kotlin functions and classes, wrapped around graphql-java.

[GitHub repo](https://github.com/ExpediaGroup/graphql-kotlin) ·
[Official website](https://opensource.expediagroup.com/graphql-kotlin/) ·
[License: Apache-2.0](https://github.com/ExpediaGroup/graphql-kotlin/blob/master/LICENSE)

## Overview

graphql-kotlin is a collection of libraries maintained by Expedia Group that run GraphQL clients and servers in Kotlin. It is built on top of `graphql-java`[^1] — the execution engine underneath is graphql-java, and graphql-kotlin is the Kotlin-idiomatic layer over it. Its defining choice is **code-first schema generation**: you write plain Kotlin classes and functions, and the schema generator uses Kotlin reflection to derive the GraphQL SDL from them at startup, rather than you authoring an SDL file and wiring resolvers to it (the schema-first approach).

The project is best understood as three loosely coupled products sharing a repo: the **schema generator** (Kotlin types → GraphQL schema, including Apollo Federation), the **servers** (Spring WebFlux and Ktor integrations that expose that schema over HTTP), and the **clients** (build-time codegen that turns `.graphql` operation files into type-safe Kotlin data classes). You can adopt any one without the others — the client generator, in particular, is useful against any GraphQL API, not just a graphql-kotlin server.

The central tension is the code-first bet. Deriving the schema from Kotlin gives you a single source of truth (the code) and free nullability mapping from Kotlin's type system, but it pushes the schema behind a reflection wall: the SDL is a build artifact of your code rather than a reviewable design document, and advanced execution behavior still requires reaching down into graphql-java. Moderately popular (~1.8k stars) and actively maintained, it is a mainstream choice for Kotlin/Spring shops that want code-first rather than the SDL-first path.

## Getting Started

Server (Spring Boot, code-first). Add the dependency:

```kotlin
// build.gradle.kts
implementation("com.expediagroup:graphql-kotlin-spring-server:8.+")
```

```kotlin
// A query resolver — the schema is generated from this class by reflection.
import com.expediagroup.graphql.server.operations.Query
import org.springframework.stereotype.Component

@Component
class BookQuery : Query {
    // Non-null Kotlin return type -> non-null GraphQL field (Book!)
    fun book(id: Int): Book = Book(id, "GraphQL in Kotlin")

    // suspend functions are supported for async resolvers
    suspend fun featured(): List<Book> = fetchFeatured()
}

data class Book(val id: Int, val title: String)
```

```yaml
# application.yml — schema generator needs the package(s) to scan
graphql:
  packages:
    - "com.example"
```

The server serves the generated schema at `/graphql` and the GraphiQL playground at `/graphiql`.

## Architecture / How It Works

**Schema generator.** The generator (`graphql-kotlin-schema-generator`) walks the Kotlin classes you register, using `kotlin-reflect` to inspect types, functions, and their signatures, and emits a `GraphQLSchema` (a graphql-java object). Kotlin functions become GraphQL fields / data fetchers; function parameters become arguments; Kotlin nullability maps directly to GraphQL nullability (`String` → `String!`, `String?` → `String`). Annotations tune the output: `@GraphQLDescription` adds schema docs, `@GraphQLIgnore` hides members, `@GraphQLName` renames, and custom hooks (`SchemaGeneratorHooks`) let you register scalars and override type resolution. Because generation happens by reflection at startup, the SDL is a runtime product of the code, not a checked-in file.

**Execution.** At request time, graphql-kotlin does very little of the actual work — `graphql-java` parses, validates, plans, and executes the query against the generated schema. This is the most important coupling to internalize: instrumentation, execution strategies, error handling semantics, and DataLoader batching are all graphql-java concepts. graphql-kotlin adds coroutine support so a `suspend` resolver is bridged to graphql-java's `CompletableFuture`-based data fetching.

**Servers.** Two server integrations exist. The Spring server (`graphql-kotlin-spring-server`) is built on Spring WebFlux (reactive/Netty), auto-configures the schema from scanned packages, and ships GraphiQL, SDL, and subscription (WebSocket) endpoints. The Ktor server plugin serves the same generated schema in a Ktor application. Both are thin — they own transport, content negotiation, and context generation, and delegate execution to graphql-java.

**Federation.** The generator can emit an Apollo Federation-compatible subgraph schema: directives like `@KeyDirective`, `@ExtendsDirective`, and the `_entities` / `_service` resolvers are generated so the service can sit behind an Apollo Router / gateway.

**Clients.** The client is a build-time story. A Gradle or Maven plugin takes your schema (an SDL file or an introspection query result) plus your `.graphql` operation files and generates type-safe Kotlin data classes for each operation, serialized with kotlinx.serialization or Jackson. At runtime a lightweight HTTP client (Ktor HTTP client or Spring `WebClient` engine) executes those operations. The generated types are the whole value proposition — the runtime is deliberately small.

## Production Notes

**Startup cost is reflection cost.** The schema is built at application startup by reflecting over your types. For large schemas this adds measurable startup latency and interacts poorly with GraalVM native-image, where reflection must be registered explicitly — native compilation of a graphql-kotlin server is possible but not the frictionless path.

**graphql-java is not optional knowledge.** The moment you need custom instrumentation, a non-default execution strategy, tracing, query complexity/depth limits, or persisted queries, you are writing graphql-java code. Teams that treat graphql-kotlin as a full framework and never learn graphql-java hit a wall on the first advanced requirement.

**N+1 and DataLoaders are manual.** `suspend` resolvers make async easy but do not batch. To avoid N+1 you register graphql-java `DataLoader`s (via the `KotlinDataLoader` interface) and resolve through them; there is no automatic batching. This is the most common production performance surprise.

**Nullability is a contract, not a suggestion.** Because Kotlin non-null types generate non-null GraphQL fields, an exception thrown in a non-null field propagates up and can null out (or error) the whole parent object per the GraphQL null-propagation rules. Designing which fields are nullable is a real schema-design decision here, made implicitly in Kotlin type declarations.

**Version coupling to Spring Boot / graphql-java.** Major versions are tied to specific Spring Boot, Kotlin, and graphql-java baselines. The Spring Boot 2 → 3 migration (the `javax` → `jakarta` namespace change) was the forcing function for a graphql-kotlin major bump, and upgrading one of these effectively pins the others. Read the release notes for the exact matrix before upgrading[^2].

**Reactive-only Spring server.** The Spring integration is WebFlux, not Spring MVC. If your service is committed to the servlet/MVC stack, the official Spring GraphQL project (spring-graphql) or DGS is a more natural fit than bending graphql-kotlin onto MVC.

**Client codegen needs the schema at build time.** The client plugin must reach the schema during the build — either a committed SDL file or a live introspection endpoint. CI without network access to the API needs the SDL vendored into the repo.

## When to Use / When Not

**Use when:**
- You want code-first GraphQL in Kotlin: schema derived from Kotlin types, with nullability and descriptions coming from the code.
- You're on Kotlin + Spring WebFlux (or Ktor) and want auto-configured server wiring.
- You need an Apollo Federation subgraph written in Kotlin.
- You want type-safe, generated Kotlin clients for any GraphQL API, decoupled from your server choice.

**Avoid when:**
- You prefer schema-first (SDL as the design artifact reviewed in PRs) — DGS or spring-graphql fit that model better.
- You're on Spring MVC / servlet stack and don't want WebFlux.
- You're building an Android or Kotlin Multiplatform client — Apollo Kotlin is purpose-built for that.
- You want to avoid startup reflection or ship a GraalVM native image with minimal fuss.

## Alternatives

- graphql-java/graphql-java — the engine graphql-kotlin sits on; use it directly when you want full control and no reflection layer, at the cost of more boilerplate.
- Netflix/dgs-framework — Spring Boot GraphQL framework on graphql-java; use it when you want schema-first (SDL) with annotation-bound resolvers and the Spring MVC stack.
- spring-projects/spring-graphql — the official Spring GraphQL integration; use it when you want schema-first plus first-class Spring support across MVC and WebFlux.
- apollographql/apollo-kotlin — client-only, Kotlin Multiplatform; use it when your problem is a GraphQL client for Android/KMP, not a server.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-09 | Repo created at Expedia Group[^3]. |
| 1.x | 2019 | Early code-first schema generator + Spring server. |
| 3.x | 2020 | graphql-java baseline bump; client generator maturing. |
| 4.x | 2021 | Spring Boot / Kotlin baseline updates. |
| 5.x | 2022 | Kotlin and Spring Boot 2.x baselines. |
| 6.x | 2023 | Spring Boot 3 / Jakarta namespace migration[^2]. |
| 7.x | 2023–2024 | graphql-java baseline bump; ongoing federation work. |
| 8.x | 2024–2025 | Current major line; Ktor client/server + federation. |

## References

[^1]: graphql-java — the underlying GraphQL execution engine. https://www.graphql-java.com/
[^2]: graphql-kotlin releases and migration notes. https://github.com/ExpediaGroup/graphql-kotlin/releases
[^3]: Repository metadata (created 2018-09-13; ~1.8k stars, ~382 forks, Apache-2.0, last push 2026-07-02), via GitHub API. https://github.com/ExpediaGroup/graphql-kotlin

## Tags

kotlin, graphql, graphql-server, graphql-client, code-first, apollo-federation, spring-webflux, ktor, schema-generator, graphql-java
