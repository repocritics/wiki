# capnproto/capnproto

> A binary serialization format whose wire layout is its in-memory layout — no parse step — plus a capability-based RPC system with promise pipelining.

[GitHub repo](https://github.com/capnproto/capnproto) ·
[Official website](https://capnproto.org) ·
[License: MIT](https://github.com/capnproto/capnproto/blob/v2/LICENSE)

## Overview

Cap'n Proto is a schema-driven serialization format and RPC system written in C++ by Kenton Varda, who was previously the primary author of Google's open-source Protocol Buffers[^1]. Its defining idea is that the encoded bytes on the wire are laid out exactly as the data structure lives in memory, so reading a message requires no decoding pass — you point a struct reader at the buffer and access fields directly by offset. The project's tagline ("infinity times faster than Protocol Buffers") is a deliberately provocative way of saying the parse step has zero cost because there is no parse step[^2].

The tradeoff is that this only holds while data stays in the compact wire format. Fields are fixed-width and word-aligned (8-byte words), variable-length data is reached through relative pointers, and reading an untrusted message means walking those pointers under bounds and traversal limits. You buy elimination of serialization CPU by accepting a rigid, alignment-sensitive layout and a security model where the reader — not the parser — is responsible for rejecting malicious inputs.

The second half of the project is RPC. Cap'n Proto RPC is capability-based: object references are first-class, and its headline feature is *promise pipelining* — you can call a method on the result of a previous call before that result has come back, so a chain of dependent calls collapses into a single network round trip. The C++ core in this repository also carries KJ, Varda's standalone C++ toolkit (async event loop, exception discipline, I/O), on which everything is built. Cap'n Proto is used heavily at Cloudflare, where Varda works; the Workers runtime (`workerd`) is built on it[^3].

## Getting Started

```bash
# macOS
brew install capnp
# Debian/Ubuntu
apt install capnproto libcapnp-dev
# or build from source — see https://capnproto.org/install.html
```

Define a schema (`addressbook.capnp`). The file ID is generated once with `capnp id`:

```capnp
@0x9eb32e19f86ee174;

struct Person {
  id    @0 :UInt32;
  name  @1 :Text;
  email @2 :Text;
}
```

Compile it, then build and read a message in C++:

```bash
capnp compile -oc++ addressbook.capnp   # emits addressbook.capnp.h/.c++
```

```cpp
#include "addressbook.capnp.h"
#include <capnp/message.h>
#include <capnp/serialize-packed.h>

capnp::MallocMessageBuilder message;
Person::Builder person = message.initRoot<Person>();
person.setId(123);
person.setName("Alice");
person.setEmail("alice@example.com");
capnp::writePackedMessageToFd(fd, message);   // packing strips zero bytes
```

The reader side does no allocation and no copy: `readPackedMessageFromFd` wraps the buffer and `getRoot<Person>()` returns a view over it.

## Architecture / How It Works

A message is a set of *segments* — flat byte arrays of 8-byte words. A struct occupies a contiguous region split into a data section (scalars, packed by size) and a pointer section (offsets to sub-objects). Because everything is offset-addressed within the arena, a whole message can be `mmap`ed from a file and used in place with no deserialization. This is what makes reads O(1) per field access rather than O(size) up front.

The schema compiler (`capnp`) parses `.capnp` files and hands an intermediate representation to language plugins (`capnpc-c++`, and third-party ones for other languages) over stdin/stdout, so adding a target language does not require touching the core compiler. Field numbers (`@0`, `@1`, ...) are permanent: schema evolution works by only ever *appending* fields and never renumbering, which is how new and old code stay wire-compatible.

Two on-wire representations exist: the raw format (fastest, larger) and the *packed* format, a trivial zero-byte-run compression that pays off because fixed-width layouts are full of zero padding. There is also a canonicalization form for signing/hashing.

RPC is layered on top. It is defined in terms of *levels*: level 1 adds promise pipelining, level 3 adds three-party capability hand-off. The C++ implementation and the Rust and Go implementations are the ones with real RPC; many other language bindings are serialization-only. The async machinery underneath is KJ's promise system, not `std::future`, and its style (explicit event loop, `kj::Promise`, optional no-exceptions mode) permeates any codebase that touches the library.

## Production Notes

**Untrusted input is the sharp edge.** Because readers follow pointers into a shared arena, a malicious message can point objects at each other to create apparent structures far larger than the actual bytes (an amplification attack). Cap'n Proto defends with a *traversal limit* (default 64 MiB of logical data read) and a *nesting depth limit*; both are enforced by `ReaderOptions`. If you parse attacker-controlled data and raise these limits carelessly, you reintroduce the DoS. Pointer validation is on by default and must stay on for untrusted input.

**Alignment is not optional.** The zero-copy design assumes messages are word-aligned. Feeding an unaligned buffer (e.g. a slice into a larger packet at an odd offset) is undefined on strict-alignment targets; the flat-array reader path expects proper alignment or a copy.

**You are adopting KJ, not just a codec.** The C++ library pulls in KJ's async and exception conventions. Teams that already have their own event loop or that build with exceptions disabled need to reconcile with KJ's model. This is a larger commitment than dropping in a header-only serializer.

**Language parity is uneven.** C++ (this repo) is the reference and the only implementation Varda maintains directly. Rust (`capnproto/capnproto-rust`) and Go (`capnproto/go-capnp`) are mature community implementations with RPC; Python (`pycapnp`) wraps the C++ core. Feature and RPC-level support differs per binding — verify before committing a polyglot system to RPC rather than plain serialization.

**Branch discipline matters.** The GitHub default branch is `v2`, an unstable development branch that may break API at any time. Production users are directed to the `master` branch, which is "1.0 plus bug fixes"[^2]. Pin to a released tarball rather than tracking `v2`.

**Build system.** Modern builds use CMake (autotools support is legacy). A recent C++ standard toolchain is required; the library targets C++14 and later.

## When to Use / When Not

**Use when:**
- Read-heavy or `mmap`-from-disk workloads where serialization CPU is the bottleneck and you want field access with zero decode cost.
- You need capability-based RPC with promise pipelining to collapse dependent round trips (its most distinctive RPC feature).
- You are already in the Cloudflare Workers / KJ / C++ orbit.

**Avoid when:**
- You need broad, first-class, officially maintained bindings across many languages — Protocol Buffers and FlatBuffers cover more ground.
- You want a small, dependency-light codec — KJ is a non-trivial dependency.
- Your data is small and infrequent; the parse cost you are eliminating does not exist at your scale, and protobuf's ecosystem (gRPC, tooling) will serve you better.
- You cannot commit to strict alignment and careful limit-tuning on untrusted input.

## Alternatives

- google/protobuf — mature, huge language and tooling ecosystem, gRPC. Use instead when you want ubiquity and a proper parse/validate step over zero-copy speed.
- google/flatbuffers — the closest peer: also zero-copy, also schema-driven, broader official language support and Google backing, but no promise-pipelining RPC. Use instead when you want zero-copy plus wide language reach.
- msgpack/msgpack — schemaless, dynamic, tiny. Use instead when you want JSON-like flexibility without a schema compiler.
- apache/thrift — serialization plus RPC across many languages. Use instead when polyglot RPC breadth outweighs pipelining.
- protocolbuffers via Cap'n Proto's own comparison: for signed/canonical small records, consider format-specific tools; Cap'n Proto's canonicalization exists but is not its core selling point.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-04 | Open-sourced by Kenton Varda; serialization core[^1]. |
| 0.4 | 2013-12 | RPC with promise pipelining introduced[^4]. |
| 0.6 | 2017-05 | Dynamic reflection, canonicalization improvements. |
| 1.0.0 | 2023-07-28 | First stable release; `master` becomes the 1.0+bugfix line[^2]. |
| v2 (branch) | ongoing | Unstable development branch; breaking changes allowed. |

## References

[^1]: Cap'n Proto home page and introduction. https://capnproto.org/
[^2]: "Cap'n Proto 1.0" release announcement — 2023-07-28. https://capnproto.org/news/2023-07-28-capnproto-1.0.html
[^3]: Cloudflare `workerd` runtime, built on Cap'n Proto. https://github.com/cloudflare/workerd
[^4]: "Time Travel! (Cap'n Proto RPC)" — promise pipelining announcement. https://capnproto.org/news/2013-12-13-promise-pipelining-capnp-vs-ice.html

## Tags

cpp, serialization, rpc, zero-copy, capability-security, schema, binary-format, promise-pipelining, ipc, cloudflare
