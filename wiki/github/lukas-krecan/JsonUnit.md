# lukas-krecan/JsonUnit

> A Java/Kotlin test library for comparing JSON documents by structure and value instead of by string equality.

[GitHub repo](https://github.com/lukas-krecan/JsonUnit) ·
[License: Apache-2.0](https://github.com/lukas-krecan/JsonUnit/blob/master/LICENSE)

## Overview

JsonUnit solves one narrow problem: asserting that two JSON documents are equal in a
test, without caring about key order, whitespace, or numeric representation. String
`assertEquals` on serialized JSON is brittle — `{"a":1,"b":2}` and `{"b":2,"a":1}` are
semantically identical but textually different — and JsonUnit exists to make that class of
false failure go away. It was first published in 2012[^1] and is still maintained, with
releases landing regularly and the `master` branch pushed within the last few days of this
writing.

The library is not an assertion framework of its own so much as an adapter layer. Its value
is that it plugs the same comparison engine into whatever assertion style a Java or Kotlin
project already uses: AssertJ (the recommended entry point), Hamcrest matchers, Spring's
`MockMvc` / `WebTestClient` / `RestTestClient` test utilities, and Kotest. All artifacts live
under the `net.javacrumbs.json-unit` Maven group and share one core comparison semantics.

The defining tension is JsonUnit's "clever" parsing of expected values. It tries to parse
the expected string as JSON; if that fails, it treats it as a plain string. This makes the
common case terse — you can write `"{a:1, b:2}"` with unquoted keys — but it produces
surprising results for primitives, where `"1"` and `"true"` get parsed as a number and a
boolean rather than strings[^2]. This footgun is the single most important thing to
understand before adopting it.

## Getting Started

AssertJ integration is the recommended API. Add the test-scoped dependency:

```xml
<dependency>
    <groupId>net.javacrumbs.json-unit</groupId>
    <artifactId>json-unit-assertj</artifactId>
    <version>6.0.1</version>
    <scope>test</scope>
</dependency>
```

```java
import static net.javacrumbs.jsonunit.assertj.JsonAssertions.assertThatJson;
import static net.javacrumbs.jsonunit.core.Option.IGNORING_ARRAY_ORDER;

// Key order and whitespace are ignored; expected value parses leniently.
assertThatJson("{\"a\":1, \"b\":2}").isEqualTo("{b:2, a:1}");

// Assert only a type, not the exact value.
assertThatJson("{\"id\": 42}").isEqualTo("{id: \"${json-unit.any-number}\"}");

// Ignore array order for one comparison.
assertThatJson("{\"a\":[1,2,3]}")
    .when(IGNORING_ARRAY_ORDER)
    .node("a").isArray().isEqualTo(new int[]{3, 2, 1});
```

## Architecture / How It Works

JsonUnit is split into a comparison core (`json-unit-core`) and thin per-framework adapter
modules. The core parses both documents into an internal node model, walks them recursively,
and emits a list of differences (missing node, extra node, value mismatch, type mismatch)
that each adapter renders into that framework's failure message.

Parsing is delegated to whichever JSON library is on the classpath — the core detects a
provider (Jackson 2, Gson, and others) rather than shipping its own parser. This keeps the
dependency footprint small in projects that already have Jackson, which is the overwhelming
common case in Spring applications. A consequence worth knowing: because Jackson is often the
provider, JsonUnit inherits Jackson's number handling and converts numeric nodes to
`BigDecimal` for comparison, which is why AssertJ map/array assertions expect `BigDecimal`
values rather than `int` or `double`.

The comparison behavior is steered by two orthogonal mechanisms:

- **Options** (`IGNORING_ARRAY_ORDER`, `IGNORING_EXTRA_FIELDS`, `TREATING_NULL_AS_ABSENT`,
  etc.), applied globally or scoped to specific paths via `when(paths(...), then(...))`.
- **Placeholders** embedded in the expected document: `${json-unit.ignore}` and
  `${json-unit.ignore-element}` to skip nodes, `${json-unit.any-string|any-number|any-boolean}`
  for type-only assertions, `${json-unit.regex}...` for pattern matching, and
  `${json-unit.matches:name}` to invoke a registered custom Hamcrest matcher.

JsonPath (Jayway) navigation is bundled inside `json-unit-core` — there is no separate
JsonPath module — so `inPath("$.store.book[0]")` and `whenIgnoringPaths("$.fields[?(@.name=='AA')].key")`
work across the AssertJ, JsonAssert, Spring, and Kotest surfaces. Hamcrest matchers support
only exact paths and the `[*]` array-index placeholder, not full JsonPath.

## Production Notes

**The lenient-parsing footgun is real.** The rule "parse expected as JSON, fall back to
string" means string-typed values that happen to look like JSON literals compare wrong.
`containsOnly("1")` against a string node `"1"` fails because `"1"` is parsed to the number
`1`; you must wrap it as `value("1")` to force string interpretation. The inverse — forcing
JSON parsing — is `json(...)`. New users lose time here; standardize on `value()`/`json()`
wrappers in shared test helpers.

**Numbers are `BigDecimal`.** Comparisons are exact by default, so `1` and `1.0` are equal
but `1.00001` is not equal to `1`. Floating-point assertions need `withTolerance(...)`.
Expect compile-time friction when passing expected numeric values to AssertJ's
`containsEntry` / `containsExactly` — they want `BigDecimal.valueOf(...)`, not primitives.

**Test-scope only.** JsonUnit has no place in production code paths; it is a test dependency
and every documented artifact is imported with `<scope>test</scope>`. Pulling it into main
scope drags a JSON provider and Hamcrest/AssertJ into your runtime for no reason.

**Version and platform baseline.** Major versions have periodically raised the minimum Java
and Spring versions, and the AssertJ API surface has grown across 4.x/5.x/6.x (Spring MockMvc
AssertJ support arrived in 4.0.0). Pin the version explicitly and read the release notes
before a major bump — the JVM baseline in particular has moved on major boundaries and can
break older build environments. (Exact minimum-Java-per-major is not restated here; verify
against the release notes for your target version.)

**Missing vs. ignored.** `${json-unit.ignore}` still requires the node to be present — it
only ignores its value; use `${json-unit.ignore-element}` or `whenIgnoringPaths(...)` when a
node may be absent entirely. Conflating the two is a common source of surprising failures.

## When to Use / When Not

**Use when:**
- You write Java or Kotlin tests that assert on JSON responses or serialized objects.
- You already use AssertJ, Hamcrest, Spring test utilities, or Kotest and want JSON-aware
  assertions in that same style.
- You need order-insensitive comparison, partial matching (`IGNORING_EXTRA_FIELDS`),
  type-only placeholders, or per-path tolerance in one library.

**Avoid when:**
- Your stack is not JVM — this is a Java/Kotlin library with no cross-language story.
- You need JSON Schema validation (contract/shape enforcement), not example-to-example
  comparison — that is a different tool.
- You want to compute and apply structured diffs/patches (RFC 6902) rather than assert
  equality in a test.

## Alternatives

- skyscreamer/JSONassert — the long-standing minimal alternative; use it when you want a
  small library with simple LENIENT/STRICT modes and no AssertJ/Kotest integration.
- rest-assured/rest-assured — use instead when your real goal is full HTTP API testing and
  JSON body assertions are just one part of it.
- json-path/JsonPath — use when you need to query/extract from JSON rather than compare two
  documents; JsonUnit actually embeds it for navigation.
- flipkart-incubator/zjsonpatch — use when you need to compute or apply JSON diffs/patches,
  not make pass/fail test assertions.
- networknt/json-schema-validator — use when you want to validate JSON against a schema
  rather than against a concrete expected example.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2012 | First published on GitHub under `net.javacrumbs.json-unit`[^1]. |
| 4.0.0 | — | Added Spring MockMvc AssertJ (`bodyJson().convertTo(jsonUnitJson())`) support[^3]. |
| 6.0.1 | current | Latest published release at time of writing; AssertJ, Hamcrest, Spring (MVC / WebTestClient / RestTestClient / REST client), and Kotest adapters[^3]. |

## References

[^1]: JsonUnit repository — created 2012-01-29, group `net.javacrumbs.json-unit`. https://github.com/lukas-krecan/JsonUnit
[^2]: JsonUnit README, "JsonUnit tries to be clever when parsing the expected value" — lenient parsing and the `value()` / `json()` wrappers. https://github.com/lukas-krecan/JsonUnit#assertj
[^3]: JsonUnit README — APIs, features, placeholders, and options. https://github.com/lukas-krecan/JsonUnit/blob/master/README.md

## Tags

java, kotlin, json, testing, assertions, assertj, hamcrest, spring, kotest, unit-testing, jvm, test-tooling
