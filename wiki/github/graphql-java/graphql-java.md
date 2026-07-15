# graphql-java/graphql-java

> The reference GraphQL execution engine for the JVM — a library, not a server.

[GitHub repo](https://github.com/graphql-java/graphql-java) ·
[Official website](https://graphql-java.com) ·
[License: MIT](https://github.com/graphql-java/graphql-java/blob/master/LICENSE.md)

## Overview

graphql-java is the oldest and most widely-embedded GraphQL implementation for
the Java Virtual Machine, started by Andreas Marek in 2015[^1]. It parses the
GraphQL schema definition language, validates queries against a schema, and
executes them against your resolver functions. That is the whole scope: it does
not ship an HTTP server, a subscriptions transport, a database mapping, or an
annotation model. Almost nobody uses it directly in application code — instead
it is the engine underneath Spring for GraphQL, Netflix DGS, and graphql-kotlin.

The defining tension is exactly that low altitude. Everything is explicit: you
build a `GraphQLSchema`, register a `DataFetcher` for every field you care
about, and wire them together through a `RuntimeWiring`. This gives total
control and no magic, at the cost of a lot of boilerplate for anything
non-trivial. Teams that want annotations, codegen, or Spring Boot autoconfig
reach for a framework on top; teams that want to understand or control every
step of execution use graphql-java directly.

The second defining trait is **breaking changes between majors**. The project
moved off semantic versioning to plain integer majors and ships roughly one
major per year, each of which may change internal APIs, execution defaults, or
the minimum JDK. Pinning a version and reading the release notes before every
upgrade is not optional here[^2].

## Getting Started

```xml
<!-- Maven -->
<dependency>
  <groupId>com.graphql-java</groupId>
  <artifactId>graphql-java</artifactId>
  <version>26.0</version>
</dependency>
```

```java
import graphql.GraphQL;
import graphql.ExecutionResult;
import graphql.schema.GraphQLSchema;
import graphql.schema.idl.*;

String sdl = "type Query { hello: String }";

TypeDefinitionRegistry registry = new SchemaParser().parse(sdl);

RuntimeWiring wiring = RuntimeWiring.newRuntimeWiring()
    .type("Query", builder -> builder
        .dataFetcher("hello", env -> "world"))
    .build();

GraphQLSchema schema =
    new SchemaGenerator().makeExecutableSchema(registry, wiring);

GraphQL graphQL = GraphQL.newGraphQL(schema).build();

ExecutionResult result = graphQL.execute("{ hello }");
System.out.println(result.getData().toString()); // {hello=world}
```

## Architecture / How It Works

The pipeline is: **parse → validate → execute**. Queries are parsed with ANTLR
into an AST (`Document`), validated against the schema (type checks, fragment
rules, variable usage), then walked by an execution strategy that invokes a
`DataFetcher` per field.

Key building blocks:

- **`GraphQLSchema`** — an immutable object graph built once at startup, either
  schema-first (SDL + `RuntimeWiring`) or programmatically
  (`GraphQLObjectType.newObject()...`). It is meant to be reused across all
  requests; rebuilding it per request is a common performance mistake.
- **`DataFetcher<T>`** — the resolver for a single field. Receives a
  `DataFetchingEnvironment` (arguments, source object, context) and returns a
  value **or** a `CompletableFuture`. If it returns a plain value it runs
  synchronously on the calling thread; returning a future is how you get
  async, non-blocking execution.
- **Execution strategies** — `AsyncExecutionStrategy` is the default and runs
  sibling fields concurrently. Mutations use `AsyncSerialExecutionStrategy` so
  top-level mutation fields run in declared order.
- **Instrumentation** — the cross-cutting hook system. Tracing, metrics, query
  depth/complexity limits, and persisted-query logic are all implemented as
  `Instrumentation` implementations that observe or short-circuit execution.
- **`ExecutionResult`** — carries `data` and `errors` together. GraphQL is
  partial-failure by design: a single field throwing produces a `null` at that
  path plus an error entry, while the rest of the response still returns.

The N+1 query problem — a list field whose child resolver hits the database once
per element — is not solved by the engine. The answer is **DataLoader**
(java-dataloader, a sibling library), which batches and caches loads within a
request. Wiring DataLoader correctly is one of the harder parts of a real
graphql-java service and is entirely on the developer.

## Production Notes

**Blocking DataFetchers block the executor.** Because a synchronous
`DataFetcher` runs on whatever thread drives execution, one blocking JDBC call
inside it stalls that thread. Under load this silently caps throughput. The fix
is to return `CompletableFuture` from any fetcher that does I/O and run it on an
appropriate thread pool — but nothing in the API forces this, so it is a
frequent latent bug.

**Query cost limiting is mandatory for public APIs.** GraphQL lets a client ask
for arbitrarily deep and wide graphs; a hostile query can be small to write and
enormous to execute. graphql-java ships `MaxQueryDepthInstrumentation` and
`MaxQueryComplexityInstrumentation`, but they are opt-in. A public endpoint
without depth/complexity limits (and ideally persisted queries) is a
denial-of-service vector.

**Schema build cost.** Parsing SDL and generating the schema is not free.
Do it once at startup and hold the `GraphQL` instance; both `GraphQLSchema` and
`GraphQL` are thread-safe and designed to be shared.

**JDK baseline creeps upward.** Early versions targeted Java 8; later majors
raised the floor to Java 11 and then Java 17. Upgrading graphql-java can force a
JVM upgrade — check the target version's requirements before bumping[^2].

**You are choosing a framework, usually.** Most teams should not adopt bare
graphql-java. Spring for GraphQL and Netflix DGS both wrap it, handle the HTTP
and subscription transports, and reduce the wiring boilerplate. Going direct is
justified when you need an unusual execution model, are building your own
framework, or want to minimize dependencies.

## When to Use / When Not

**Use when:**
- You are on the JVM and need GraphQL server-side execution.
- You are building a framework or platform and want the raw engine to control.
- You need precise control over execution strategy, instrumentation, or schema
  construction that a higher-level framework hides.

**Avoid when:**
- You want annotations, codegen, and Spring Boot autoconfiguration out of the
  box — use DGS or Spring for GraphQL (which use this underneath anyway).
- You are not on the JVM — use the implementation native to your stack.
- You want a ready-made GraphQL server over an existing database without writing
  resolvers.

## Alternatives

- graphql/graphql-js — the JavaScript reference implementation; use when your stack is Node, not the JVM.
- Netflix/dgs-framework — Spring Boot framework built on graphql-java; use when you want annotations, codegen, and less wiring.
- spring-projects/spring-graphql — the official Spring integration, also on graphql-java; use when you are already in the Spring ecosystem.
- expediagroup/graphql-kotlin — code-first schema generation for the JVM; use when you are in Kotlin and want the schema derived from your types.
- hasura/graphql-engine — auto-generated GraphQL over Postgres; use when you want a server over a database instead of writing resolvers.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-07 | Repository created; first GraphQL implementation for the JVM, by Andreas Marek[^1]. |
| — | ~2018 | Moved to plain integer major versions; roughly yearly major cadence begins. |
| — | later majors | JDK baseline raised from Java 8 → 11 → 17; internal API and execution-default changes between majors[^2]. |
| 26.0 | 2026-04-23 | Latest release at time of writing[^3]. |

## References

[^1]: graphql-java README and repository history. Copyright 2015, Andreas Marek and contributors. https://github.com/graphql-java/graphql-java
[^2]: graphql-java releases and changelog. https://github.com/graphql-java/graphql-java/releases
[^3]: graphql-java v26.0 release, published 2026-04-23 (GitHub Releases API). https://github.com/graphql-java/graphql-java/releases/latest

## Tags

java, jvm, graphql, graphql-server, api, schema, execution-engine, library, spring, backend
