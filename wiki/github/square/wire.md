# square/wire

> A protobuf and gRPC toolchain for the JVM/Kotlin/Swift world that ships its own schema compiler instead of wrapping `protoc`.

[GitHub repo](https://github.com/square/wire) ·
[Official website](https://square.github.io/wire/) ·
[License: Apache-2.0](https://github.com/square/wire/blob/master/LICENSE.txt)

## Overview

Wire is Square's implementation of Protocol Buffers and gRPC, targeting Android, Kotlin, Swift, and Java[^1]. First published in 2013, its original pitch was pragmatic: Google's `protobuf-java` generated large classes with builders and heavy runtime reflection, which mattered on Android where method count and APK size are budgeted. Wire generates smaller, immutable message classes with a compact runtime.

The defining architectural choice is that Wire does **not** shell out to the `protoc` binary. It contains its own `.proto` parser and schema model (`wire-schema`), written in Kotlin, and runs code generation entirely on the JVM through a Gradle plugin[^2]. This removes the native `protoc` toolchain from the build, makes generation hermetic and cacheable, and lets Wire offer schema-level operations `protoc` does not — most notably pruning unused messages and fields out of generated code. The tradeoff: Wire is a second, independent implementation of the spec, so edge cases, newer proto features, and option handling can diverge from the reference compiler, and Wire deliberately omits parts of the spec it considers not worth the weight.

Wire 3 (2019) was a ground-up rewrite from Java to Kotlin that added Kotlin data-class output, Swift generation, and a coroutine-based gRPC client (`wire-grpc`) built on OkHttp rather than Netty[^3]. That lineage — small runtime, Android-first, OkHttp transport — still distinguishes Wire from the mainline gRPC stack today.

## Getting Started

Wire is consumed as a Gradle plugin; there is no standalone binary to install.

```kotlin
// build.gradle.kts
plugins {
  id("com.squareup.wire") version "5.0.0"
}

wire {
  kotlin {}          // emit Kotlin; use java {} or swift {} for other targets
}
```

Put schemas under `src/main/proto`:

```protobuf
// src/main/proto/person.proto
syntax = "proto2";
package example;

message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;
}
```

Wire generates an immutable `Person` with a `ProtoAdapter` for wire-format encode/decode:

```kotlin
val person = Person(name = "Alice", id = 42)
val bytes: ByteArray = Person.ADAPTER.encode(person)
val decoded: Person = Person.ADAPTER.decode(bytes)
```

## Architecture / How It Works

Wire is three layers that are usually used together but are separable:

1. **`wire-schema`** — a pure-Kotlin `.proto` parser and linker. It builds an in-memory schema model, resolves imports, and applies `roots`/`prunes` rules. Because it is plain JVM code, it is also usable as a library for schema tooling, linting, and migrations independent of code generation.
2. **Code generators** — pluggable emitters for Kotlin, Java, and Swift. Kotlin output uses KotlinPoet; message classes are immutable and carry a generated `ADAPTER` companion of type `ProtoAdapter<T>` that does the serialization. There is no runtime reflection over field descriptors on the hot path.
3. **`wire-runtime` / `wire-grpc`** — the small runtime the generated code depends on, plus the gRPC client. `wire-grpc` uses OkHttp's HTTP/2 support for transport and exposes service calls as coroutine `suspend` functions or `Flow`s.

Generation is driven entirely by the Gradle plugin, so the build graph — not a separate `protoc` invocation — owns proto compilation. This is what makes `prune` practical: you declare the message roots your app actually reaches and Wire drops everything transitively unreachable, which is meaningful on Android where a large shared schema would otherwise inflate the binary.

Wire supports both proto2 and proto3, and can read and re-emit schemas (useful for schema migration and codegen across languages). JSON encoding is available through Moshi or Gson adapters rather than a built-in reflective JSON path.

## Production Notes

- **`wire-grpc` is a client-first, lightweight stack.** It rides on OkHttp, which is excellent for Android and JVM clients, but it is not a drop-in replacement for `grpc-java`'s full server ecosystem: interceptors, name resolvers, load-balancing policies, and the broad plugin surface of the mainline stack are not all present. If you are building a heavily-featured gRPC *server*, evaluate `grpc-java` first.
- **Wire is a second implementation, not a `protoc` wrapper.** Newly added protobuf language features, exotic options, and custom-option handling may lag or differ from the reference compiler. Teams sharing schemas across a `protoc`-based backend and a Wire-based Android client should confirm both compilers agree on the wire format for the features they use.
- **Pruning is a footgun as well as a feature.** `prune` silently removes types; an over-aggressive root/prune configuration can drop fields still present on the wire, which then decode as unknown fields rather than failing loudly.
- **Required fields (proto2).** Wire honors proto2 `required`, and decoding a message missing a required field throws. Long-lived schemas that started with `required` inherit the well-known protobuf rigidity around evolving those fields.
- **Version cadence.** The Java-to-Kotlin jump at 3.0 changed generated API shape (Java builders → Kotlin data-class copy semantics), so 2.x → 3.x was a real migration, not a drop-in upgrade[^3]. Later 4.x/5.x releases track Kotlin, coroutines, and KMP target changes; pin the plugin version and read the changelog before crossing majors[^4].

## When to Use / When Not

**Use when:**
- You target Android/Kotlin/Swift and care about generated-code size and a small runtime.
- You want protobuf codegen without a native `protoc` toolchain in the build.
- You need Kotlin Multiplatform or Swift message models from a shared schema.
- You want coroutine-native gRPC clients on OkHttp.

**Avoid when:**
- You need a full-featured gRPC server with the mainline interceptor/LB ecosystem — use `grpc-java`.
- You depend on the very latest protobuf spec features or exact `protoc` parity across a polyglot backend.
- Your stack is outside JVM/Kotlin/Swift (Go, C++, Python, …), where the reference implementations are the natural fit.

## Alternatives

- protocolbuffers/protobuf — the reference implementation; use it when you need full spec coverage, `protoc` parity, or non-JVM/Swift language targets.
- grpc/grpc-java — use it when you need a full gRPC server with the complete interceptor, resolver, and load-balancing ecosystem Wire's OkHttp client does not match.
- grpc/grpc-kotlin — use it when you want coroutine-based gRPC but on top of `grpc-java`/`protobuf-java` rather than Wire's independent runtime.
- apple/swift-protobuf — use it for Swift protobuf when you are not sharing a single schema toolchain with Kotlin/Java.
- google/flatbuffers — use it when zero-copy deserialization matters more than protobuf schema ergonomics.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013 | Initial release; Android-focused Java protobuf codegen with a small runtime[^1]. |
| 2.0 | 2016 | Java-generation era; compact messages, own schema parser. |
| 3.0 | 2019 | Kotlin rewrite; Kotlin/Swift output, `wire-grpc` on OkHttp, coroutines[^3]. |
| 4.x | 2021+ | Kotlin Multiplatform targets, continued gRPC and schema tooling work[^4]. |
| 5.x | 2024+ | Current major line; see changelog for target/runtime changes[^4]. |

## References

[^1]: Wire README and project site. https://square.github.io/wire/
[^2]: Wire Gradle plugin and schema model documentation. https://square.github.io/wire/wire_compiler/
[^3]: "Wire 3.0" — Kotlin rewrite, Swift, gRPC on OkHttp. https://square.github.io/wire/
[^4]: Wire changelog. https://square.github.io/wire/changelog/

## Tags

kotlin, protobuf, grpc, protocol-buffers, android, swift, code-generation, serialization, rpc, jvm, square
