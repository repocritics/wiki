# vavr-io/vavr-jackson

> Jackson datatype module that teaches `ObjectMapper` to (de)serialize Vavr's persistent collections, tuples, and control types.

[GitHub repo](https://github.com/vavr-io/vavr-jackson) ·
[Vavr website](https://vavr.io/) ·
[License: Apache-2.0](https://github.com/vavr-io/vavr-jackson/blob/main/LICENSE)

## Overview

Vavr (formerly Javaslang, renamed in 2017[^1]) is a functional library for Java 8+ that
adds persistent immutable collections (`List`, `HashMap`, `TreeSet`, …) and functional
control types (`Option`, `Either`, `Try`, `Lazy`, `Tuple0`–`Tuple8`). Out of the box
Jackson does not understand any of these types — it falls back to bean introspection and
produces wrong or unusable JSON. `vavr-jackson` is the adapter module that registers the
serializers and deserializers so `io.vavr.collection.List<Integer>` round-trips as a plain
JSON array `[1,2,3]` instead of leaking Vavr's internal `head`/`tail` structure.

It is a thin, single-purpose module: one `VavrModule` you add to an `ObjectMapper`, plus a
small `Settings` object for the handful of behaviors where JSON has no canonical mapping
(chiefly how `Option` is encoded). There is no runtime of its own — it lives or dies with
Jackson and Vavr, and its job is to make the two agree.

The defining tension is versioning. The module's **major version tracks Jackson's major
version, not Vavr's**: `vavr-jackson 0.x` targets Jackson 2.x (the `com.fasterxml.jackson`
package namespace), while `vavr-jackson 1.x` targets Jackson 3.x (the relocated
`tools.jackson` namespace)[^2]. Both lines are built against Vavr 0.x. Picking the wrong
one is the single most common integration mistake, because the two Jackson namespaces are
not interchangeable and a mismatch surfaces as `NoClassDefFoundError` rather than a clear
message.

## Getting Started

```xml
<!-- Maven — Jackson 3.x -->
<dependency>
  <groupId>io.vavr</groupId>
  <artifactId>vavr-jackson</artifactId>
  <version>1.0.0</version>
</dependency>
```

```java
import io.vavr.collection.List;
import io.vavr.control.Option;
import io.vavr.jackson.datatype.VavrModule;
import tools.jackson.core.type.TypeReference;   // Jackson 3 namespace
import tools.jackson.databind.ObjectMapper;
import tools.jackson.databind.json.JsonMapper;

ObjectMapper mapper = JsonMapper.builder()
    .addModule(new VavrModule())
    .build();

String json = mapper.writeValueAsString(List.of(1, 2, 3));   // "[1,2,3]"

// Element type MUST come through a TypeReference — raw types erase it:
List<Integer> restored =
    mapper.readValue(json, new TypeReference<List<Integer>>() {});   // List(1, 2, 3)
```

On Jackson 2.x, use `vavr-jackson:0.10.x` and the `com.fasterxml.jackson.*` imports instead.

## Architecture / How It Works

`VavrModule` extends Jackson's `Module` and, in its `setupModule` hook, installs a
`Serializers` and `Deserializers` pair plus type modifiers. For collections it maps each
Vavr type to the closest JSON shape: sequences and sets become JSON arrays, maps and
multimaps become JSON objects (or arrays of entries when keys aren't string-coercible),
and tuples become fixed-length arrays. Deserialization keys off the declared generic type,
which is why a `TypeReference` (or a typed field on a bean) is mandatory — Vavr's own
generics are erased at runtime and the module cannot recover the element type otherwise.

Control types are where the design choices are visible. `Option` is serialized in a
**plain format by default**: `Some(x)` becomes `x` and `None` becomes `null`. This is the
ergonomic choice for interop with clients that expect a nullable field, but it is lossy —
`Some(null)` and `None` both encode to `null` and cannot be told apart on the way back. The
alternative tagged format (`["defined",42]` / `["undefined"]`), enabled with
`Settings.useOptionInPlainFormat(false)`, is unambiguous but bespoke. `Either` and `Tuple`
have no ambiguity and always use array encoding.

Configuration is a single immutable-style `VavrModule.Settings` builder passed to the
constructor. The knobs that exist are exactly the ones where JSON underdetermines the
mapping: `useOptionInPlainFormat`, `deserializeNullAsEmptyCollection` (turn a JSON `null`
into an empty Vavr collection rather than `null`), and a couple of related null-handling
flags. There is deliberately no per-type customization surface beyond this.

## Production Notes

- **Version-to-Jackson coupling is the top footgun.** `1.x` = Jackson 3 (`tools.jackson`),
  `0.x` = Jackson 2 (`com.fasterxml.jackson`). If your app or a transitive dependency pins
  Jackson 2, you must stay on `vavr-jackson 0.10.x`; the namespaces do not coexist cleanly.
- **`Option` plain format loses `Some(null)`.** If null-inside-Some is a meaningful state in
  your domain, switch to the tagged format. Most APIs are fine with plain, but the default
  silently collapses the distinction.
- **Forgetting to register the module fails quietly-ish.** Without `addModule(new
  VavrModule())`, Jackson tries bean introspection on Vavr types and emits structurally
  wrong JSON (or throws on deserialization) rather than telling you the module is missing.
- **Generic erasure on read.** `readValue(json, List.class)` yields a raw `List` and mangled
  element types; always use `new TypeReference<List<T>>(){}` or a typed target field.
- **Maintenance is quiet by design.** The module is a thin adapter and its commit velocity
  is low, but it is not abandoned — activity clusters around Jackson major releases (the
  recent work targets the Jackson 3 / `1.0` line). Vavr core itself has sat on the `0.10.x`
  line for a long time with a long-promised `1.0` still pending, so do not expect this
  module to move faster than its upstream.
- **No `Try`/`Future`/`Validation` serialization guarantees.** Coverage centers on
  collections, tuples, `Option`, `Either`, `Lazy`, and functions; effectful/async control
  types are not a natural JSON fit — verify before relying on them.

## When to Use / When Not

**Use when:**
- Your codebase already uses Vavr and you need its types to cross a JSON boundary (REST
  APIs, Kafka payloads, config) without hand-writing converters.
- You want immutable collections in your DTOs and want them to serialize as ordinary arrays
  and objects.
- You need `Option` at the wire edge and want a plain nullable encoding for free.

**Avoid when:**
- You are not otherwise committed to Vavr — adding the whole functional library just to get
  Jackson support for immutable collections is a large dependency for a small need; use a
  lighter datatype module instead.
- You are on Jackson 2 and cannot upgrade, and you want the latest `1.x` features — you are
  capped at the `0.10.x` line.
- You need serialization of async/effect types (`Future`, `Try`) as a first-class contract.

## Alternatives

- google/guava with `jackson-datatype-guava` — immutable collections (`ImmutableList`,
  `ImmutableMap`) and Jackson support without the functional control types; use when you
  want persistence-ish immutability but not Vavr.
- FasterXML/jackson-modules-java8 (`jackson-datatype-jdk8`) — use when the only Vavr type
  you actually serialize is `Option`; JDK `Optional` covers it with a first-party module.
- eclipse/eclipse-collections — richer immutable/primitive collections with its own Jackson
  integration; use when memory density and primitive specialization matter more than
  functional control types.
- arrow-kt/arrow — the Kotlin equivalent (`Option`, `Either`, immutable data); use instead
  when your stack is Kotlin rather than Java.
- Plain FasterXML/jackson-databind with JDK collections — use when you can drop Vavr from
  your DTO layer entirely and keep functional types internal to the service.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2015-10-30 | Started as `javaslang-jackson`[^3]. |
| — | 2017 | Javaslang renamed to Vavr; module became `vavr-jackson`, groupId `io.vavr`[^1]. |
| 0.10.x | — | Long-lived stable line, tracks Vavr 0.10.x on Jackson 2.x. |
| 1.0.0 | — | Jackson 3.x support (`tools.jackson` namespace); still built on Vavr 0.x[^2]. |

## References

[^1]: "Javaslang is now Vavr" — the library and its modules were renamed in early 2017; groupId moved from `io.javaslang` to `io.vavr`. https://vavr.io/
[^2]: Vavr-Jackson README, version compatibility matrix (Jackson 2.x → vavr-jackson 0.x; Jackson 3.x → vavr-jackson 1.x). https://github.com/vavr-io/vavr-jackson
[^3]: Repository metadata, GitHub API (`created_at` 2015-10-30). https://github.com/vavr-io/vavr-jackson

## Tags

java, jackson, json, serialization, vavr, functional-programming, immutable-collections, datatype-module, jvm, apache-2.0
