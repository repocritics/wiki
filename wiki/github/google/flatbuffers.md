# google/flatbuffers

> A cross-language serialization library that lets you read structured data without a parse or unpack step.

[GitHub repo](https://github.com/google/flatbuffers) ·
[Official website](https://flatbuffers.dev/) ·
[License: Apache-2.0](https://github.com/google/flatbuffers/blob/master/LICENSE)

## Overview

FlatBuffers is a schema-driven serialization library originally written by Wouter van Oortmerssen at Google and open-sourced in 2014[^1]. It came out of Android game development, where the cost that mattered was not wire size but the CPU and allocation spent turning bytes back into objects on load. Its defining property is zero-copy access: the serialized buffer *is* the in-memory representation, and generated accessors read fields directly out of it at their byte offsets. There is no unpack step, no per-object allocation, and no eager traversal — you can `mmap` a multi-megabyte buffer and touch only the handful of fields you actually read.

You define data in a `.fbs` schema, run the `flatc` compiler to generate accessor code for your target languages, and use the generated `FlatBufferBuilder` to write buffers. Forward and backward compatibility comes from per-table vtables: every table stores a small offset table, so readers written against an older or newer schema find the fields they know and ignore the rest. This is the same design goal as Protocol Buffers[^2] (also Google), but the tradeoff is inverted — FlatBuffers pays with a larger, alignment-padded wire format and a more awkward write API in exchange for effectively free reads.

The library is stable and widely embedded (TensorFlow Lite's model format is FlatBuffers, as are many game and networking stacks), but it is in low-churn maintenance mode rather than active feature expansion. The write ergonomics remain its most persistent source of friction, and for schema-less use it ships a separate format, FlexBuffers.

## Getting Started

Build the `flatc` compiler (there is no single universal binary; most package managers also carry it):

```bash
cmake -G "Unix Makefiles"
make -j
# or: brew install flatbuffers  /  apt install flatbuffers-compiler
```

Define a schema:

```fbs
// monster.fbs
namespace Game;

table Monster {
  name:string;
  hp:short = 100;
  pos:Vec3;
}

struct Vec3 { x:float; y:float; z:float; }

root_type Monster;
```

Generate accessors, then build and read a buffer in C++:

```bash
./flatc --cpp monster.fbs   # writes monster_generated.h
```

```cpp
#include "monster_generated.h"
flatbuffers::FlatBufferBuilder b;
auto name = b.CreateString("Orc");       // children first...
auto mon  = Game::CreateMonster(b, name, 80);
b.Finish(mon);                           // ...root last

const uint8_t *buf = b.GetBufferPointer();
auto m = Game::GetMonster(buf);          // zero-copy, no parse
printf("%s %d\n", m->name()->c_str(), m->hp());
```

## Architecture / How It Works

A FlatBuffer is built **bottom-up and back-to-front**. Because a parent table stores offsets to its children, the children must exist in the buffer before the parent can reference them, and the builder grows the buffer from the end toward the front. This is why the API forces you to create strings, vectors, and nested tables *first* and pass their offsets into the parent constructor last — a constraint that surprises almost everyone on first contact and is the root of most "why won't this compile" questions.

The wire format has three primitive categories:

- **structs** — fixed-size, inlined, no vtable. Fast and compact but frozen: you can never add or remove a field without breaking the format. Use only for truly stable value types (coordinates, colors).
- **tables** — the evolvable unit. Each table instance points to a **vtable** that maps field IDs to byte offsets; absent fields have offset 0 and fall back to their schema default, costing zero bytes. Vtables are deduplicated within a buffer, so many similar tables share one.
- **vectors, strings, unions** — length-prefixed, offset-referenced heap-like regions.

Everything is aligned to its natural boundary, so the format inserts padding. That padding, plus the vtables and 32-bit offsets, is why FlatBuffers on the wire is typically larger than the equivalent Protobuf.

`flatc` is the code generator and also a runtime tool: it can convert JSON to/from binary against a schema, emit a binary schema (`.bfbs`) for reflection, and generate for C++, C, C#, Dart, Go, Java, Kotlin, Lua, Lobster, PHP, Python, Rust, Swift, TypeScript and more. Each language has a hand-written runtime library that the generated code sits on top of. **FlexBuffers** is a companion, schema-less format for cases where you don't want to compile a schema at all — self-describing like JSON, still zero-copy, but without field-level evolution guarantees.

## Production Notes

- **Untrusted input must be verified.** Zero-copy means the accessors trust the offsets in the buffer. A malformed or malicious buffer can point offsets anywhere; reading it without first running the generated `Verifier` is a memory-safety bug, not just a data error. The verifier is a separate, explicit call — it is easy to forget, and forgetting it on network-facing input is the single most dangerous FlatBuffers footgun. OSS-Fuzz runs continuously against the library[^3], which is a direct consequence of this design.
- **Mutation is limited.** You can mutate scalar fields of an existing buffer in place (`mutate_hp()`), but you cannot add fields, grow strings/vectors, or change structure without rebuilding the whole buffer. Treat buffers as read-mostly.
- **Schema evolution has strict rules.** New fields may only be added at the *end* of a table's field list, and fields are `deprecated`, never deleted or reordered — the field order defines the vtable slots. Break this and old readers misread new buffers. `id:` attributes let you manage ordering explicitly.
- **Larger wire size than Protobuf.** Alignment padding and vtables cost bytes. If your bottleneck is bandwidth or on-disk footprint rather than decode CPU, Protobuf or a compressed format often wins. FlatBuffers pays off when the same buffer is read many times, or read partially, or memory-mapped.
- **Write path is the weak side.** The builder API is verbose and the ordering constraints are unforgiving across most languages. Codebases that both write and read hot data sometimes keep Protobuf on the write side and FlatBuffers only where zero-copy reads matter.
- **Versioning is date-based.** The project abandoned SemVer for a `YY.MM.DD` release scheme[^4], so a version number tells you *when* a release shipped, not whether it contains breaking changes — read release notes rather than inferring compatibility from the number.

## When to Use / When Not

**Use when:**
- You read the same serialized data many times, or read only a few fields out of a large record — `mmap` + zero-copy is the whole point.
- Decode latency or per-load allocation is your measured bottleneck (games, in-vehicle/embedded, ML model files, high-throughput RPC).
- You need cross-language, evolvable data with no parse step on the read side.

**Avoid when:**
- Wire size or bandwidth is the constraint — the padding/vtable overhead works against you.
- You write far more than you read, or need to mutate/grow data after serialization — the builder friction and immutability hurt.
- You want ergonomics and a schema-optional workflow — JSON, MessagePack, or CBOR are simpler; use FlexBuffers if you want zero-copy without a schema.

## Alternatives

- protocolbuffers/protobuf — use when smaller wire size and a nicer write API matter more than zero-copy reads; requires a parse step.
- capnproto/capnproto — closest philosophical peer (also zero-copy, by a former Protobuf maintainer); use when you also want a built-in RPC system.
- msgpack/msgpack — use when you want compact, schema-less, JSON-like encoding and don't need zero-copy or field evolution.
- google/flatbuffers (FlexBuffers) — the same repo's schema-less format; use when you want zero-copy reads but can't or won't compile a schema.
- Apache Avro — use in data/analytics pipelines where dynamic schema resolution and Hadoop-ecosystem integration matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-06 | Open-sourced by Google; C++, Java, Go, and `flatc` [^1]. |
| 1.x | 2015–2020 | Language coverage broadened; FlexBuffers added; gRPC integration. |
| 2.0.0 | 2021-05 | Major release; consolidated 64-bit/large-buffer and generator changes. |
| date-based | 2023+ | Switched to `YY.MM.DD` versioning (e.g. 23.x, 24.x, 25.x) [^4]. |

## References

[^1]: Wouter van Oortmerssen, "FlatBuffers: a memory efficient serialization library" — Android Developers Blog, 2014-06-17. https://android-developers.googleblog.com/2014/06/flatbuffers-memory-efficient.html
[^2]: FlatBuffers documentation, "FlatBuffers vs. Protocol Buffers" comparison. https://flatbuffers.dev/flatbuffers_benchmarks.html
[^3]: OSS-Fuzz continuous fuzzing, flatbuffers project. https://bugs.chromium.org/p/oss-fuzz/issues/list?q=proj:flatbuffers
[^4]: FlatBuffers wiki, "Versioning" rationale. https://github.com/google/flatbuffers/wiki/Versioning

## Tags

serialization, zero-copy, cpp, cross-platform, code-generation, schema, protobuf-alternative, mmap, rpc, flatbuffers
