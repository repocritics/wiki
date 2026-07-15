# networknt/json-schema-validator

> A Jackson-based Java implementation of JSON Schema (Draft 4 through 2020-12), built for runtime request/response validation in the light-4j microservices stack.

[GitHub repo](https://github.com/networknt/json-schema-validator) ·
[Documentation](https://doc.networknt.com/library/json-schema-validator/) ·
[License: Apache-2.0](https://github.com/networknt/json-schema-validator/blob/master/LICENSE)

## Overview

`json-schema-validator` is a Java validator for the JSON Schema Core specification, covering Draft 4, 6, 7, 2019-09, and 2020-12[^1]. It parses schemas and instance documents with Jackson, which is its defining design choice: if a project already uses Jackson (most Java web stacks do), the validator adds no second JSON parser and no impedance mismatch. It reports 100% pass on the required and optional portions of the JSON Schema Test Suite across all supported drafts, using the Joni regex factory for `pattern`/`format` tests[^2].

The library exists because it is a load-bearing dependency of networknt's [light-4j](https://github.com/networknt/light-4j) framework, where it validates HTTP requests and responses against OpenAPI specifications (via light-rest-4j) and RPC schemas (via light-hybrid-4j) on every call at runtime[^1]. That origin explains the project's priorities: throughput and allocation behavior are treated as first-order concerns, and the maintainers publish JMH benchmarks against competing implementations[^3]. It also explains the release discipline problem below — the library evolves in lockstep with the framework, and API breaks land in minor versions.

The central tension is scope. This is not a lightweight "is-this-JSON-valid" helper; it is a full, cross-dialect validation engine with meta-schema validation, custom dialect/vocabulary/keyword support, OpenAPI 3.0/3.1 dialects, and pluggable regex and schema-retrieval backends. That completeness is the reason to choose it, and the configuration surface is the reason to read the docs before shipping.

## Getting Started

Two release lines are published to Maven Central, differing only by Jackson and JDK baseline[^1]:

- **2.x** — Java 8+, Jackson 2.x
- **3.x** — Java 17+, Jackson 3.x

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>json-schema-validator</artifactId>
    <version>2.0.4</version>
</dependency>
```

```java
// Register a dialect as the default when the schema omits $schema.
SchemaRegistry registry = SchemaRegistry.withDefaultDialect(
        SpecificationVersion.DRAFT_2020_12,
        builder -> builder.schemaIdResolvers(r ->
                r.mapPrefix("https://example.com/schema", "classpath:schema")));

Schema schema = registry.getSchema(
        SchemaLocation.of("https://example.com/schema/example-main.json"));

List<Error> errors = schema.validate(input, InputFormat.JSON, ctx ->
        // Since Draft 2019-09, `format` is annotation-only unless you opt in.
        ctx.executionConfig(cfg -> cfg.formatAssertionsEnabled(true)));
```

The same `validate` path handles YAML input (`InputFormat.YAML`) and validating a schema against a dialect's meta-schema — the bundled meta-schemas for Drafts 4–2020-12 are loaded from the classpath by default[^1].

## Architecture / How It Works

A `SchemaRegistry` is the entry point. It holds dialect configuration, schema-retrieval strategy, and a `SchemaRegistryConfig`. Calling `getSchema` resolves and compiles a schema into a tree of keyword validators; `schema.validate(...)` walks the instance against that tree and returns a `List<Error>` (an empty list means valid). Schema compilation is separable from validation, so the intended usage is compile-once, validate-many — building the `Schema` per request throws away the main optimization.

Schema resolution is indirection-heavy by design. `$id` values are mapped to retrieval IRIs through `schemaIdResolvers` (e.g. `mapPrefix`), and retrieval IRIs are resolved to bytes through configurable loaders — classpath, a `Map<String,String>` of URI→schema, or an arbitrary `Function<String,String>`. This lets you bundle schemas as resources and keep `$ref` resolution fully offline, which is the sane default for a validator embedded in a service.

Regex handling is pluggable and is the sharpest edge. By default the library uses the JDK's `java.util.regex`, which is **not** ECMA 262 compliant, so `pattern` and regex-`format` behavior diverges from what a spec-strict validator would do. Two opt-in factories exist: `GraalJSRegularExpressionFactory` (best compliance, ~50 MB of GraalVM JS dependencies) and `JoniRegularExpressionFactory` (the Oniguruma engine, ~2 MB)[^1]. Selecting a factory without adding its optional dependency yields a `ClassNotFoundException` at runtime, not a compile error.

Dependencies are deliberately minimal: slf4j-api, jackson-databind, jackson-dataformat-yaml, and ethlo `itu` (for RFC 3339 `date`/`date-time`). The YAML and `itu` dependencies are excludable via Maven `<exclusions>` if you never validate YAML or need only `OffsetDateTime`-grade date parsing[^1]. Custom dialects, vocabularies, keywords, and formats are all extension points[^1].

## Production Notes

**Annotation collection is the hidden performance cliff.** `unevaluatedProperties` and `unevaluatedItems` force the sibling `properties`/`items` validators to collect annotations, and that collection materially slows validation[^1]. If a hot-path schema uses these keywords, benchmark it — throughput is highly workload-dependent and the maintainers explicitly warn that their published numbers only reflect one Draft 4 `properties`-heavy workload[^3].

**Deeply nested `oneOf`/`anyOf` without a short-circuit is both slow and unreadable.** With no `if`/`then` to prune branches, the engine evaluates every alternative, and a failure returns the union of every child's error — often a confusing wall of messages[^1]. Structure discriminated unions with `if`/`then`/`else` where possible.

**`format` is annotation-only by default from Draft 2019-09 onward.** A schema that says `"format": "email"` will silently not assert unless `formatAssertionsEnabled(true)` is set on the `SchemaRegistryConfig` or per-execution `ExecutionConfig`[^1]. This trips up teams migrating from Draft 7, where `format` asserted.

**Minor versions can break you.** The project states plainly that minor releases may contain breaking changes requiring code edits, and maintains a dedicated upgrading document for that reason[^1]. Pin exact versions, read the upgrade notes, and do not use version ranges. The API surface itself was reshaped in the modern line (the `SchemaRegistry`/`Schema`/`Error` API shown here supersedes the older `JsonSchemaFactory`/`JsonSchema`/`ValidationMessage` API from earlier 1.x releases), so upgrading across that boundary is a code migration, not a version bump.

**Regex default is a correctness footgun, not just a compliance note.** If your schemas were authored against ECMA 262 semantics (they usually are), the JDK regex default can accept or reject strings the schema author did not intend. Choosing Graal or Joni is the safe path for anything that must match spec behavior; weigh the ~50 MB Graal footprint against the 2 MB Joni option.

## When to Use / When Not

**Use when:**
- Your service already uses Jackson and you want validation without a second parser.
- You need current drafts (2019-09 / 2020-12) and OpenAPI 3.0/3.1 dialect validation.
- You're inside or adjacent to the light-4j / networknt ecosystem.
- You need custom keywords, vocabularies, or offline `$ref` resolution.

**Avoid when:**
- You want a dependency you can upgrade blindly — minor-version breaks make that unsafe.
- You need a tiny, zero-config validator for a single fixed draft.
- Strict ECMA 262 regex is mandatory but you cannot add the Graal/Joni dependency.
- You're not on the JVM — this is Java-only.

## Alternatives

- everit-org/json-schema — long-established JVM validator; uses org.json rather than Jackson and historically topped out around Draft 7. Use when you want a mature, stable API and don't need 2019-09/2020-12.
- leadpony/justify — streaming validator built on the Jakarta JSON-P API. Use when you validate large documents incrementally and prefer standard JSON-P over Jackson.
- harrel-io/json-schema — newer, spec-focused implementation with 2020-12 support and provider-agnostic JSON binding. Use when you want a clean modern API and pluggable JSON providers.
- java-json-tools/json-schema-validator — the original fge validator. Effectively unmaintained and Draft-4-era; use only for legacy code you cannot migrate.
- python-jsonschema/jsonschema or ajv-validator/ajv — reach for these when your stack is Python or JavaScript rather than the JVM.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2016-09 | Created as a light-4j component, Draft 4 focus[^4]. |
| 1.0.x line | 2017–2023 | Multi-draft support added (6, 7, 2019-09, 2020-12); `JsonSchemaFactory` API. |
| 2.x line | current | Java 8+/Jackson 2.x; redesigned `SchemaRegistry` API[^1]. |
| 3.x line | current | Java 17+/Jackson 3.x parallel release line[^1]. |

Exact per-release dates are on the GitHub Releases page[^5]; the two active lines (2.x and 3.x) are maintained in parallel and differ only by JDK/Jackson baseline.

## References

[^1]: networknt/json-schema-validator README. https://github.com/networknt/json-schema-validator
[^2]: JSON Schema Test Suite results (README functional table; uses `JoniRegularExpressionFactory`). https://github.com/json-schema-org/JSON-Schema-Test-Suite
[^3]: JSON Schema Validator Perftest (JMH) and Creek JVM validator comparison. https://www.creekservice.org/json-schema-validation-comparison/performance
[^4]: Repository metadata: created 2016-09-15, Apache-2.0, ~1.07k stars, 347 forks as of 2026-07 (GitHub API).
[^5]: Releases. https://github.com/networknt/json-schema-validator/releases

## Tags

java, json-schema, validation, jackson, openapi, jvm, draft-2020-12, schema-validation, light-4j, data-validation
