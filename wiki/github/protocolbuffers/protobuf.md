# protocolbuffers/protobuf

> Google's schema-first binary serialization format and compiler — the IDL that most of the RPC world (gRPC included) is built on.

[GitHub repo](https://github.com/protocolbuffers/protobuf) ·
[Official website](https://protobuf.dev) ·
License: BSD-3-Clause[^1]

## Overview

Protocol Buffers ("protobuf") is a language- and platform-neutral mechanism for
serializing structured data, open-sourced by Google in 2008[^2]. You define
messages in a `.proto` schema, run the `protoc` compiler to generate accessor
classes in your target language, and exchange a compact tag-length-value binary
wire format between programs. It is the on-the-wire format and interface
definition language for gRPC, and the default serialization layer inside much of
Google's own infrastructure.

The defining tradeoff is schema-first rigidity in exchange for size, speed, and
cross-language/forward-backward compatibility. Unlike JSON or MessagePack, you
cannot parse a protobuf payload without the schema — the wire bytes carry field
*numbers*, not field *names*. That constraint is what buys the compatibility
guarantees (add fields freely, old readers skip unknown ones) but also what makes
protobuf operationally heavier: every consumer needs the compiled schema, and the
generated code, `protoc`, and runtime library versions all have to stay within a
supported compatibility window[^3].

This repository is the C++ implementation: `protoc` itself, the C++ runtime, plus
the Java, Python, C#, Ruby, PHP, and Objective-C runtimes. Go, Dart, and
JavaScript live in separate repos (`protocolbuffers/protobuf-go`,
`dart-lang/protobuf`, `protocolbuffers/protobuf-javascript`)[^4].

## Getting Started

```bash
# macOS
brew install protobuf
# Debian/Ubuntu
apt-get install -y protobuf-compiler
# or download a prebuilt protoc-$VERSION-$PLATFORM.zip from GitHub releases
protoc --version   # e.g. libprotoc 35.1
```

```proto
// person.proto — proto3 syntax
syntax = "proto3";
package example;

message Person {
  string name = 1;         // field NUMBER 1 is the wire identity, not "name"
  int32  id   = 2;
  optional string email = 3;  // proto3 field presence (since 3.15)
}
```

```bash
protoc --python_out=. --cpp_out=. person.proto
# generates person_pb2.py and person.pb.{h,cc}
```

```python
from person_pb2 import Person
p = Person(name="Ada", id=42)
data = p.SerializeToString()   # compact binary
p2 = Person(); p2.ParseFromString(data)
```

## Architecture / How It Works

Three pieces move independently:

1. **`protoc`** — a C++ compiler that parses `.proto` files into an in-memory
   `FileDescriptor` graph, then invokes per-language **code generator plugins**
   (built-in for C++/Java/Python/etc., or external `protoc-gen-*` binaries
   discovered on `PATH`). Plugins receive a serialized `CodeGeneratorRequest`
   (itself a protobuf) and emit generated source.
2. **The wire format** — each field is encoded as a varint tag `(field_number <<
   3 | wire_type)` followed by its value. There are only a handful of wire types
   (varint, 64-bit, length-delimited, 32-bit). Unknown fields are preserved as
   raw bytes on parse, which is what makes forward compatibility work. There is
   no field-name or type information on the wire — the schema is mandatory to
   decode.
3. **Runtime libraries** — per language, providing generated-message base
   classes, reflection, descriptors, and the well-known types (`Any`,
   `Timestamp`, `Duration`, `Struct`, `FieldMask`, wrappers).

The historically important split is **proto2 vs proto3**. proto2 had explicit
`required`/`optional`/`repeated` and default values; proto3 dropped `required`
and (initially) scalar field presence, treating unset scalars as their zero
value. That "no presence" decision was reversed — `optional` scalars returned in
3.15 (2021)[^5]. The current direction is **Editions**, which replaces the
proto2/proto3 syntax fork with a single model where behaviors (field presence,
enum openness, UTF-8 validation) are controlled by per-file/per-field *features*
that evolve edition-to-edition[^6]. Edition 2023 was the first.

Bazel is the canonical build system (Bzlmod on Bazel 8+); CMake is supported for
the C++ runtime. The README explicitly warns that even release *branches* can be
unstable between release commits, and that source builds should pin to a release
tag[^7].

## Production Notes

- **Never reuse or renumber a field.** Field numbers are the wire identity.
  Deleting a field and later assigning its number to a new field silently
  corrupts data for readers with the old schema. Use `reserved 3, 5 to 7;` and
  `reserved "old_name";` to fence off retired numbers/names.
- **`required` (proto2) is a known design mistake.** A required field can never be
  safely removed — every reader will reject a message missing it, so you can't
  stage a schema change across services. Avoid it; proto3 removed it for this
  reason.
- **proto3 zero-vs-unset ambiguity.** Without `optional`, a scalar set to `0`,
  `""`, or `false` is indistinguishable from unset and is *not serialized*. This
  bites APIs that need to express "clear this field." Use `optional`, wrapper
  types, or `FieldMask`.
- **Version skew is real in C++.** Generated `.pb.h`/`.pb.cc` must match the
  linked runtime version; mismatches produce link errors or ABI crashes. protoc
  and runtimes follow a documented cross-version support policy[^3], and the
  version numbers were *decoupled per language* in the 2022 release (C++/protoc
  went from 3.21 to 21.x, Python to 4.21, etc.)[^8] — so "protobuf 25" means
  different library versions in different languages.
- **JSON mapping has sharp edges.** 64-bit integers serialize as JSON *strings*;
  `Any` requires a type registry; unknown fields and default values have
  configurable but easy-to-miss handling.
- **No built-in schema registry.** Distributing `.proto` files, linting them, and
  detecting breaking changes is left to you or third-party tooling — `bufbuild/buf`
  is the de facto modern layer for module management, lint, and breaking-change
  detection on top of protobuf.
- **Generated code is large.** Especially C++ and Java; message-heavy schemas
  produce substantial binary and compile-time cost.

## When to Use / When Not

**Use when:**
- You control both producers and consumers (or can distribute schemas) and want
  compact, fast, evolvable messages across languages.
- You are building gRPC services — protobuf is its native IDL.
- You need forward/backward schema compatibility with real guarantees.

**Avoid when:**
- You need self-describing payloads readable without a schema — reach for JSON,
  CBOR, or MessagePack.
- You want zero-copy access to fields without a parse/allocate step — FlatBuffers
  or Cap'n Proto fit better.
- Your data pipeline needs dynamic schemas and a registry-first workflow (Kafka,
  data lakes) — Avro is the ecosystem default there.

## Alternatives

- google/flatbuffers — use when deserialization cost matters more than message
  size; access fields in-place with no parse step.
- capnproto/capnproto — use when you want zero-copy plus an RPC system with
  promise pipelining.
- apache/avro — use when you need dynamic schemas and a schema registry for
  data-pipeline / streaming workloads rather than RPC.
- msgpack/msgpack — use when you want compact binary encoding without an IDL or
  code generation step (schemaless, JSON-like).
- apache/thrift — use when you want a single project bundling IDL, serialization,
  and RPC transport/server stack together.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2008-07 | Open-sourced by Google; proto2 semantics[^2]. |
| 3.0.0 | 2016-07 | proto3 syntax, more languages, canonical JSON mapping. |
| 3.15 | 2021-02 | `optional` field presence returns to proto3[^5]. |
| 21.0 | 2022-05 | Per-language version decoupling (C++/protoc drop the "3." prefix)[^8]. |
| ~26 | 2024 | Editions (Edition 2023) begin replacing the proto2/proto3 fork[^6]. |
| 35.0 | 2026-05 | Recent stable major (35.1 as of 2026-06). |
| 36.0-rc1 | 2026-07 | Latest release-candidate at time of writing[^9]. |

## References

[^1]: The repository LICENSE is a 3-clause BSD license (Copyright 2008 Google LLC). GitHub's license detector reports `NOASSERTION` because of the custom copyright header, but the terms are BSD-3-Clause. https://github.com/protocolbuffers/protobuf/blob/main/LICENSE
[^2]: Protocol Buffers documentation, history/overview. https://protobuf.dev/overview/
[^3]: Protobuf version support policy. https://protobuf.dev/version-support/
[^4]: README language/runtime table. https://github.com/protocolbuffers/protobuf#protobuf-runtime-installation
[^5]: Field presence in proto3 (`optional`). https://protobuf.dev/programming-guides/field_presence/
[^6]: Protobuf Editions overview. https://protobuf.dev/editions/overview/
[^7]: README, "Working With Protobuf Source Code" (pin to release commits). https://github.com/protocolbuffers/protobuf#working-with-protobuf-source-code
[^8]: Protobuf versioning change (decoupled per-language version numbers). https://protobuf.dev/support/version-support/
[^9]: GitHub Releases (v36.0-rc1, 2026-07-09; v35.1, 2026-06-11). https://github.com/protocolbuffers/protobuf/releases

## Tags

serialization, protobuf, protoc, idl, binary-format, grpc, cplusplus, code-generation, wire-format, schema-evolution, rpc
