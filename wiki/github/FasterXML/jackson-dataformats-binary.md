# FasterXML/jackson-dataformats-binary

> Umbrella project holding Jackson streaming/databind backends for five binary formats: Avro, CBOR, Ion, Protobuf, and Smile.

[GitHub repo](https://github.com/FasterXML/jackson-dataformats-binary) ·
[License: Apache-2.0](https://github.com/FasterXML/jackson-dataformats-binary/blob/master/LICENSE)

## Overview

This repository is not a library so much as a collection of five sibling libraries that share one thing: they let you read and write a binary wire format using the same Jackson API you already use for JSON — streaming (`JsonParser`/`JsonGenerator`), databinding (`ObjectMapper`), and tree model[^1]. Each backend subclasses Jackson's core abstractions so that swapping `new JsonFactory()` for `new CBORFactory()` changes the bytes on the wire without changing your data-binding code. It sits under the broader Jackson project (whose JSON core has been the default JVM serialization stack for over a decade) and moves in lockstep with it.

The defining tension is that the five formats have almost nothing in common except this shared adapter surface. CBOR and Smile are self-describing, schema-free binary encodings that behave like a denser JSON — you can point them at arbitrary POJOs and they just work. Avro and Protobuf are schema-first formats designed around an external contract; the Jackson backends bolt a POJO-mapping convenience layer on top of formats that were never meant to be driven by reflection, and the impedance mismatch shows up as feature gaps and setup friction. Ion (Amazon's format) sits in between. Treating "jackson-dataformats-binary" as one uniform thing is the most common way teams get surprised.

Maintenance is volunteer-only. The author is Tatu Saloranta (@cowtowncoder); the Ion backend is maintained by Michael Liedtke[^2]. There is no paid support and no SLA on issue response, which matters when you hit a format edge case.

## Getting Started

Each format is a separate Maven artifact; pull only the one you need. Versions must match your `jackson-databind` version exactly.

```xml
<dependency>
  <groupId>com.fasterxml.jackson.dataformat</groupId>
  <artifactId>jackson-dataformat-cbor</artifactId>
  <version>2.18.1</version>
</dependency>
```

`[FORMAT]` is one of `avro`, `cbor`, `ion`, `protobuf`, `smile`.

```java
// CBOR: schema-free, drop-in replacement for a JSON ObjectMapper
CBORMapper mapper = new CBORMapper();          // added in 2.10 for convenience
byte[] encoded = mapper.writeValueAsBytes(myPojo);
MyPojo back = mapper.readValue(encoded, MyPojo.class);
```

```java
// Protobuf: schema-first — you must supply a schema before reading/writing
ProtobufMapper mapper = new ProtobufMapper();
ProtobufSchema schema = mapper.generateSchemaFor(MyPojo.class);  // or parse a .proto
byte[] encoded = mapper.writer(schema).writeValueAsBytes(myPojo);
MyPojo back  = mapper.readerFor(MyPojo.class).with(schema).readValue(encoded);
```

## Architecture / How It Works

Every backend follows the same skeleton: a `JsonFactory` subclass (`CBORFactory`, `SmileFactory`, `AvroFactory`, `ProtobufFactory`, `IonFactory`) that produces format-specific `JsonParser` and `JsonGenerator` implementations, plus — since 2.10 — a dedicated `ObjectMapper` subclass and Builder-style construction[^1]. Databind and tree model sit unchanged on top, because they only ever talk to the streaming abstraction. This is why the same annotations, modules, and `ObjectMapper` configuration carry across formats.

Where the formats diverge is everything below the streaming API:

- **CBOR** (RFC 7049 / RFC 8949) and **Smile** (Jackson's own format) are token-for-token mappable to the JSON data model. They are self-describing, need no schema, and support the full JSON type set plus binary and typed numbers. Smile predates CBOR and is Jackson-specific; CBOR is an IETF standard and the better default for interop.
- **Avro** wraps Apache's `avro` library. It needs an Avro schema, which the backend can generate from a POJO or parse from `.avsc`. Coverage of Avro's own features (logical types, unions, schema evolution rules) is partial and has historically lagged the underlying spec — this is the backend most likely to reveal a gap.
- **Protobuf** implements the wire format directly and drives it from a `ProtobufSchema` (generated from POJOs or parsed from `.proto`). It is not a gRPC stack and does not aim for full parity with Google's `protobuf-java`; some proto3 semantics and well-known types are not modeled.
- **Ion** delegates to Amazon's `ion-java` and supports both binary and text Ion, including Ion-specific types (timestamps, annotations, symbols) that have no JSON equivalent.

The schema-free formats (CBOR, Smile) are the ones that behave like "JSON but smaller/faster." The schema-bearing formats (Avro, Protobuf, Ion) require you to reason about an external contract and accept that the reflection-driven mapping is a convenience layer, not a full-fidelity binding.

## Production Notes

**Version lockstep is non-negotiable.** A format backend at 2.17 with `jackson-databind` at 2.18 will produce `NoSuchMethodError`/`LinkageError` at runtime, not a clean failure. Manage all `com.fasterxml.jackson.*` artifacts through `jackson-bom` and never let a transitive dependency pin an older core.

**Binary parsers are a hardened attack surface.** The project is enrolled in Google OSS-Fuzz[^3], and binary decoders have historically been a rich source of resource-exhaustion CVEs across the Jackson ecosystem — malformed length prefixes triggering huge allocations, deeply nested documents causing stack exhaustion. Jackson core added `StreamReadConstraints` (2.15, tightened since) to cap nesting depth, string/number/name length, and document size; these limits apply to the binary backends too. If you decode untrusted bytes, do not raise those limits without understanding the memory blast radius, and keep the whole Jackson stack patched.

**Avro is the sharpest edge.** Schema generation from POJOs does not cover every annotation combination, logical-type support has been incomplete, and behavior can differ from the reference Apache Avro tooling. If you need strict schema-evolution guarantees or full logical-type fidelity, generate schemas explicitly and validate cross-tool rather than trusting POJO-inferred schemas.

**Protobuf is not wire-identical to `protobuf-java` in every case.** It reads and writes the format, but if another service uses generated protobuf stubs and relies on proto3 defaults, oneofs, maps, or well-known types, verify round-trips against the real `.proto` toolchain before committing to it. There is no gRPC integration here.

**Smile locks you in.** It is compact and fast but Jackson-specific — nothing outside the JVM ecosystem reads it. Choose CBOR when any non-Java consumer might touch the data.

**Jackson 3.0 is a hard break.** The `master`/`3.x` line moves to a new `tools.jackson` package namespace and API changes; it is not a drop-in upgrade from 2.x. Most production users should stay on the 2.x line (2.18/2.19) until 3.x is a deliberate migration[^4].

## When to Use / When Not

**Use when:**
- You already serialize with Jackson and want a smaller/faster wire format without rewriting binding code (CBOR or Smile).
- You need to interoperate with an existing Avro, Protobuf, or Ion pipeline from Java and prefer POJO binding over code generation.
- You want streaming, databind, and tree-model access to a binary format from one API.

**Avoid when:**
- You need full-fidelity Protobuf/gRPC — use Google's `protobuf-java` and generated stubs.
- You need strict Avro schema evolution or complete logical-type support — use the official Apache Avro Java library.
- You are on a non-JVM stack, or the data crosses many languages — pick a standard format (CBOR) with first-class native libraries, not Smile.

## Alternatives

- protocolbuffers/protobuf — use the official `protobuf-java` with generated stubs when you need full Protobuf/gRPC fidelity instead of reflection-based mapping.
- apache/avro — use the reference Avro Java library when schema evolution and logical types must be exactly correct.
- msgpack/msgpack-java — use MessagePack (it ships its own Jackson backend) when you want a widely-supported cross-language binary format Jackson doesn't host here.
- amzn/ion-java — use Ion directly when you want the format's full feature set without the Jackson binding layer.
- FasterXML/jackson-dataformats-text — sibling umbrella for text formats (CSV, YAML, TOML, Properties) when your target isn't binary.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.8 | 2016-07 | Repo consolidated the previously separate per-format modules; oldest history here[^5]. |
| 2.10 | 2019-09 | Per-format `ObjectMapper` subclasses (`CBORMapper`, `SmileMapper`, …) and Builder-style construction added[^1]. |
| 2.15 | 2023-04 | `StreamReadConstraints` limits applied across backends to bound untrusted decoding. |
| 2.18 | 2024-09 | Current stable 2.x line at time of writing (2.18.1). |
| 2.19 | 2025 | Next 2.x development branch. |
| 3.0 | in progress | `tools.jackson` namespace, breaking API changes; developed on `master`/`3.x`[^4]. |

## References

[^1]: Project README — module structure, API styles, and 2.10 mapper/builder additions. https://github.com/FasterXML/jackson-dataformats-binary/blob/master/README.md
[^2]: README "Maintainers" — Tatu Saloranta (@cowtowncoder), Michael Liedtke (@mcliedtke, Ion); volunteer maintenance. https://github.com/FasterXML/jackson-dataformats-binary#maintainers
[^3]: Google OSS-Fuzz continuous fuzzing project list. https://github.com/google/oss-fuzz/tree/master/projects/jackson-dataformats-binary
[^4]: Jackson 3.0 release plan and `tools.jackson` namespace change. https://github.com/FasterXML/jackson-future-ideas/wiki/Jackson-3.0
[^5]: README "Branches" — history before 2.8 lived in separate per-format repositories. https://github.com/FasterXML/jackson-dataformats-binary#branches

## Tags

java, jvm, serialization, binary-format, jackson, cbor, smile, avro, protobuf, ion, data-format, streaming-parser
