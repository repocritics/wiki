# skyscreamer/JSONassert

> A Java assertion library that compares two JSON documents by logical structure rather than by string, so tests survive key reordering and formatting changes.

[GitHub repo](https://github.com/skyscreamer/JSONassert) ·
[Official website](http://jsonassert.skyscreamer.org) ·
[License: Apache-2.0](https://github.com/skyscreamer/JSONassert/blob/master/LICENSE.txt)

## Overview

JSONassert is a small test-scoped library for JVM projects that turns "does this JSON match what I expected?" into a single `assertEquals` call. You pass an expected JSON string and an actual JSON value; internally both are parsed into `org.json` objects and compared field-by-field, so a difference in whitespace, key order, or (optionally) extra fields does not fail the test. When a mismatch is found it produces a path-addressed diff message like `friends[id=3].pets[]: Expected bird, but not found`, which is the feature most users actually stay for — hand-written `org.json` assertions produce nothing comparable.

The library's defining choice is the strictness axis. `JSONAssert.assertEquals(expected, actual, strict)` takes a boolean, and the maintainers explicitly recommend `false` (lenient): expected fields must all be present and equal, but the actual document may contain extra keys and may order array elements and object keys differently[^1]. This makes REST-interface tests resilient to backend additions, at the cost of silently passing when the server drops a field you never asserted on. That tradeoff — brittleness vs. permissiveness — is the entire design conversation around JSONassert, and choosing the wrong mode is the most common source of tests that are either flaky or falsely green.

It is deliberately narrow: it does JSON equality assertions and nothing else. No HTTP client, no schema validation, no path querying. This is why it has stayed relevant since 2012[^2] despite minimal feature growth — it is a leaf dependency that other tools (Spring's test support, hamcrest-json) wrap rather than replace.

## Getting Started

```xml
<!-- Maven, test scope -->
<dependency>
    <groupId>org.skyscreamer</groupId>
    <artifactId>jsonassert</artifactId>
    <version>1.5.1</version>
    <scope>test</scope>
</dependency>
```

```java
import org.skyscreamer.jsonassert.JSONAssert;
import org.skyscreamer.jsonassert.JSONCompareMode;

String actual = callRestEndpoint("/friends/367.json");
String expected = "{friends:[{id:123,name:\"Corby Page\"},{id:456,name:\"Carter Page\"}]}";

// lenient: order-insensitive, extra keys in actual are allowed
JSONAssert.assertEquals(expected, actual, false);

// strict: exact — no extra keys, arrays must be in order
JSONAssert.assertEquals(expected, actual, JSONCompareMode.STRICT);
```

Note the expected string uses relaxed JSON (unquoted keys) — `org.json` accepts it, which keeps test literals readable.

## Architecture / How It Works

Comparison is driven by `JSONCompare`, which walks the expected tree and checks each node against the actual tree. Behavior is governed by two independent booleans exposed as the `JSONCompareMode` enum: `STRICT`, `LENIENT`, `NON_EXTENSIBLE`, and `STRICT_ORDER`[^3]. The two axes are *extensibility* (may the actual document contain keys the expected one does not?) and *array order* (must array elements match by position?). The four named modes are just the four combinations; the `strict` boolean overload maps to `STRICT`/`LENIENT`.

Array matching in lenient mode is the subtle part. When order is not enforced, JSONassert tries to pair elements between the two arrays, and for arrays of objects it heuristically matches by a unique key so that diff messages can say `friends[id=3]` instead of `friends[2]`. This pairing is combinatorial, which matters for large arrays (see Production Notes).

Customization is the escape hatch. `CustomComparator` takes a set of `Customization` rules, each binding a JSON path (e.g. `friends[*].timestamp`) to a `ValueMatcher`. Built-ins include `RegularExpressionValueMatcher` and `ArrayValueMatcher`, and you can implement `ValueMatcher<Object>` for things like "this field just has to be a valid ISO date" or "ignore this generated id." This is how teams assert on responses that contain non-deterministic values without falling back to raw parsing.

The biggest structural fact is the JSON backend. Through 1.x, JSONassert depended not on the canonical `org.json` but on `com.vaadin.external.google:android-json`, a repackaged Android implementation, specifically to avoid the JSON.org license's "shall be used for Good, not Evil" clause, which many corporate legal teams reject[^4]. As of the 2.0 line, JSONassert switched to stleary/JSON-java as its `org.json` provider[^5]. This is invisible in normal use but changes the transitive dependency and the exact parsing quirks.

## Production Notes

**Lenient mode hides dropped fields.** `assertEquals(expected, actual, false)` only checks that expected keys exist and match. If your server stops returning a field you don't assert on, the test stays green. For contract tests where completeness matters, use `NON_EXTENSIBLE` or `STRICT` and assert the full document.

**Large unordered arrays are slow.** Lenient array matching without a unique key degenerates toward O(n²) pairing. Tests over arrays of hundreds/thousands of objects can become noticeably slow or produce confusing "best-effort" diffs. Mitigations: enforce `STRICT_ORDER` when the order is actually deterministic, or add a `Customization` that identifies elements by id.

**The 2.0 release has been a release candidate for a long time.** The current published artifact is `2.0-rc1`; many teams still pin `1.5.1` for production builds because it is the last final release[^6]. Treat 2.0 as usable but understand you are depending on an RC, and pin the exact version.

**Number and type coercion can surprise you.** Because comparison goes through `org.json`, `1` vs `1.0` vs `"1"` are not always interchangeable, and very large integers or high-precision decimals can round-trip differently than your source JSON. Assert on strings when exact numeric fidelity matters.

**Maintenance is low-tempo.** The repository is not archived but sees infrequent commits and long gaps between releases[^2]. It is stable and widely embedded (Spring Boot's test support ships it as `JsonContentAssert`'s engine), so "quiet" here means "done," not "dead" — but do not expect fast turnaround on new features or issues.

**Kotlin/Android note.** It works fine on the JVM generally, but the string-literal-heavy API is verbose in Kotlin, and on Android the 1.x `android-json` dependency can collide with the platform's own `org.json`.

## When to Use / When Not

**Use when:**
- You are unit/integration testing REST or messaging endpoints that return JSON and want order-insensitive, whole-or-partial document assertions.
- You want readable, path-addressed diffs instead of hand-rolled `org.json` navigation.
- You need to ignore or pattern-match a handful of non-deterministic fields via custom comparators.

**Avoid when:**
- You want to assert individual values by path rather than compare documents — JsonPath fits better.
- You need JSON *schema* validation (types, required, formats) — this is not a schema validator.
- You want rich placeholder/diff ergonomics and Jackson/Gson integration — JsonUnit is more capable there.
- You are testing full HTTP behavior (status, headers, body) end-to-end — reach for REST Assured, using JSONassert only for the body if needed.

## Alternatives

- lukas-krecan/JsonUnit — use when you want placeholders, regex/ignore tokens, better diffs, and native Jackson/Gson/AssertJ integration; the most common "grew out of JSONassert" upgrade.
- json-path/JsonPath — use when you assert specific extracted values (`$.friends[0].id`) instead of comparing whole documents.
- hertzsprung/hamcrest-json — use when your suite is Hamcrest-matcher based; it wraps JSONassert's comparison in `sameJSONAs(...)` matchers.
- rest-assured/rest-assured — use when you are testing the HTTP endpoint end-to-end (status, headers, body) rather than only comparing JSON strings.
- stleary/JSON-java — not an alternative but the underlying `org.json` engine; use directly when you need low-level parsing, not assertions.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-02 | First public commit; authored around Carter Page and Solomon Duskis[^2]. |
| 1.2.x | 2013–2014 | `JSONCompareMode`, custom comparators / `ValueMatcher`. |
| 1.5.0 | 2017 | Widely adopted stable line; embedded in Spring Boot test support. |
| 1.5.1 | 2022 | Last final 1.x release; still the common production pin[^6]. |
| 2.0-rc1 | 2022+ | Switches `org.json` provider to stleary/JSON-java; long-lived RC[^5]. |

## References

[^1]: JSONassert README — lenient mode recommendation and error-message example. https://github.com/skyscreamer/JSONassert/blob/master/README.md
[^2]: GitHub repository metadata — created 2012-02-01, last pushed 2024-07-28. https://github.com/skyscreamer/JSONassert
[^3]: `JSONCompareMode` Javadoc — STRICT / LENIENT / NON_EXTENSIBLE / STRICT_ORDER. http://jsonassert.skyscreamer.org/apidocs/index.html
[^4]: JSON.org license "good, not evil" clause and the `android-json` workaround, background. https://www.json.org/license.html
[^5]: JSONassert README, "org.json" section — v2 uses stleary/JSON-java. https://github.com/skyscreamer/JSONassert#orgjson
[^6]: Maven Central — org.skyscreamer:jsonassert versions. https://mvnrepository.com/artifact/org.skyscreamer/jsonassert

## Tags

java, testing, json, unit-testing, rest, assertions, junit, jvm, test-tooling, api-testing
