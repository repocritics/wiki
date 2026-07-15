# alibaba/fastjson

> A fast Java JSON library whose speed came from autoType — the same feature that made it a decade-long RCE liability. Now archived in favor of fastjson2.

[GitHub repo](https://github.com/alibaba/fastjson) ·
[Upgrade guide](https://github.com/alibaba/fastjson2/wiki/fastjson_1_upgrade_cn) ·
[License: Apache-2.0](https://github.com/alibaba/fastjson/blob/master/license.txt)

## Overview

Fastjson is a Java library for serializing Java objects to JSON and parsing
JSON back into objects, created by Wenshao (温少) at Alibaba and public since
2011[^1]. For most of the 2010s it was the default JSON library in a large
share of Chinese-market Java systems, chosen for raw throughput: it generated
per-class serializers as JVM bytecode via ASM rather than reflecting field by
field, and it consistently placed near the top of the Eishay `jvm-serializers`
benchmark.

The defining tension is that fastjson's speed and its worst liability come from
the same design. To deserialize polymorphic fields, fastjson supported
**autoType** — a JSON payload could carry an `@type` key naming an arbitrary
Java class, and fastjson would instantiate it and invoke its setters. That made
generic deserialization convenient, and it also turned any endpoint that parsed
untrusted JSON into a gadget-chain remote-code-execution surface. From roughly
2017 onward fastjson lived through a public arms race of autoType bypasses and
blacklist patches (see Production Notes); this is what the library is best known
for outside Alibaba.

As of 2026 the repository is **archived and read-only**[^2]. Its own
description tells you to leave: the 1.x line is end-of-life at 1.2.83, and
active development moved to the ground-up rewrite [fastjson2](https://github.com/alibaba/fastjson2).
This page documents fastjson 1.x — treat it as legacy you are migrating off,
not a library you adopt.

## Getting Started

Fastjson 1.x is on Maven Central as `com.alibaba:fastjson`. The `1.2.x`
coordinates are the frozen legacy line; the `2.0.x` coordinates under the same
artifact id are a **compatibility bridge** that delegates to fastjson2, intended
only to ease migration.

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.83</version>
</dependency>
```

```java
import com.alibaba.fastjson.JSON;

public class Demo {
    static class User { public int id; public String name; }

    public static void main(String[] args) {
        User u = new User();
        u.id = 1; u.name = "Tom";

        String json = JSON.toJSONString(u);          // {"id":1,"name":"Tom"}
        User back  = JSON.parseObject(json, User.class);

        // DO NOT do this on untrusted input — autoType surface:
        // Object o = JSON.parseObject(json, Object.class);
    }
}
```

For any new code, depend on `com.alibaba.fastjson2:fastjson2` and use its
`com.alibaba.fastjson2.JSON` entrypoint instead.

## Architecture / How It Works

The serializer path (`JSON.toJSONString`) compiles a dedicated
`ObjectSerializer` per class using ASM to emit bytecode that writes fields
directly to a `SerializeWriter` buffer. This code generation is the reason
fastjson benchmarked faster than reflection-based libraries — the cost is paid
once per type, then amortized.

The parser (`DefaultJSONParser` + a hand-written lexer) walks JSON tokens and
binds them to a target type. When the caller does not pin a concrete type — or
when a payload contains an `@type` field and `autoTypeSupport` is enabled — the
parser resolves the named class through an `ParserConfig` and constructs it.
Because setters and certain constructors run during construction, a class whose
instantiation has side effects (JNDI lookups, template evaluation, JDBC
datasource wiring) becomes an exploitation primitive. The historical defense was
a **blacklist** of dangerous class-name prefixes inside `ParserConfig`; each
published bypass added a new prefix, which is a losing structure against an open
class universe.

Version 1.2.68 (late 2020) introduced **safeMode**, which stops honoring
`@type` entirely regardless of blacklist state — a whitelist/deny-by-default
posture rather than deny-by-list[^3]. safeMode is the correct setting for any
1.x deployment that must survive, but it is *off by default*, so upgrading the
jar alone does not close the hole.

fastjson2 restructured this: autoType is opt-in and gated by an explicit
allow-list (`Filter` / context-supplied whitelist) from the start, and the
serialization engine was rewritten for JDK 8+/records and higher throughput.
That is why the maintainers chose a new artifact and a hard cut rather than a
point release.

## Production Notes

- **safeMode is not the default.** On fastjson 1.x you must set it explicitly:
  `ParserConfig.getGlobalInstance().setSafeMode(true)`, or the
  `-Dfastjson.parser.safeMode=true` system property, or the
  `fastjson.properties` entry. Confirm it is on in every JVM; a single service
  parsing untrusted JSON without it is exploitable.
- **Blacklist patching is not a security strategy.** Numerous autoType bypasses
  were disclosed across 1.2.x; CVE-2022-25845 is the widely-cited one affecting
  versions before 1.2.83[^4]. Chasing point releases without safeMode leaves you
  one gadget class away from RCE.
- **`@type` in stored data.** Some systems persisted fastjson output (cache,
  message queues, DB columns) that includes `@type`. After enabling safeMode or
  migrating, that stored data may fail to deserialize; audit for round-tripped
  `@type` before flipping the switch.
- **Behavioral quirks vs. other libraries.** fastjson is lenient by default —
  it tolerates non-standard JSON, orders map keys differently, and has its own
  date-format and `BigDecimal`/`long` handling. Migrating to Jackson/Gson or to
  fastjson2 can surface subtle output differences in field ordering, numeric
  precision, and null handling; diff serialized output in tests, do not assume
  drop-in equivalence.
- **The 2.0.x compatibility bridge is a migration aid, not a destination.**
  Pointing `com.alibaba:fastjson` at `2.0.x` gets you fastjson2's engine under
  the old API surface, but the API is not identical and some edge behaviors
  change. Plan to move to the `fastjson2` API directly.
- **Archived means no fixes.** Since the repo is read-only, no new 1.x security
  patch will ship. Any 1.x jar you run today is frozen at its last release.

## When to Use / When Not

**Use when:**
- You are maintaining an existing 1.x codebase and cannot migrate immediately —
  in which case enable safeMode today and schedule the move to fastjson2.

**Avoid when:**
- You are starting anything new — use fastjson2 or a maintained alternative.
- The service parses untrusted or externally-influenced JSON and you cannot
  guarantee safeMode is enforced everywhere.
- You need a library with an active security-response process; this repo is
  archived.

## Alternatives

- alibaba/fastjson2 — the maintainer-blessed successor; use this instead in
  virtually every case (new code and migrations alike).
- FasterXML/jackson-databind — the de facto Java JSON standard; use when you want
  the broadest ecosystem, Spring integration, and an active security track.
- google/gson — use when you want a smaller, simpler, reflection-based library
  without codegen and without autoType-style polymorphism by default.
- eclipse-vertx/vertx or jsoniter — use jsoniter when raw parse speed is the
  priority and you can accept a smaller community.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-11 | Public on GitHub; ASM-based codegen serializer[^1]. |
| autoType era | ~2017 onward | Repeated autoType-bypass RCE disclosures and blacklist patches. |
| 1.2.68 | 2020-11 | safeMode introduced — disables `@type` regardless of blacklist[^3]. |
| 1.2.83 | 2022-05 | Final 1.x release; addresses CVE-2022-25845[^4]. |
| fastjson2 | 2022 | Ground-up rewrite in a new artifact/repo; autoType opt-in + allow-list. |
| archived | 2024 | Repository set read-only; upgrade guidance points to fastjson2[^2]. |

## References

[^1]: fastjson repository and README. https://github.com/alibaba/fastjson
[^2]: Repository archived (read-only) with description recommending fastjson2. https://github.com/alibaba/fastjson2
[^3]: fastjson safeMode documentation (autoType disable). https://github.com/alibaba/fastjson/wiki/enable_autotype
[^4]: CVE-2022-25845 — fastjson autoType bypass deserialization RCE affecting versions before 1.2.83. https://nvd.nist.gov/vuln/detail/CVE-2022-25845

## Tags

java, json, serialization, deserialization, security, rce, deprecated, alibaba, jvm, legacy
