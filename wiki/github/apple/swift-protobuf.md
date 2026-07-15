# apple/swift-protobuf

> Apple's official protoc plugin and runtime library for using Protocol Buffers from Swift, with value-type generated code and full Google conformance.

[GitHub repo](https://github.com/apple/swift-protobuf) ·
[Protocol Buffers](https://protobuf.dev/) ·
[License: Apache-2.0](https://github.com/apple/swift-protobuf/blob/main/LICENSE.txt)

## Overview

Swift Protobuf is two things shipped from one repository: `protoc-gen-swift`, a
code-generation plugin for Google's `protoc` compiler, and `SwiftProtobuf`, the
runtime library the generated code depends on[^1]. You write a `.proto` schema,
run `protoc --swift_out`, and get a `.pb.swift` file per input; the generated
types are ordinary Swift `struct`s with copy-on-write value semantics, `Equatable`,
and `Hashable` conformance. It is maintained by Apple and tracks the language
closely — recent releases require a Swift 6.1+ toolchain[^2].

The defining design choice is that generated messages are value types, not
classes. This is idiomatic Swift and avoids a class of aliasing bugs, but it
diverges from every other official protobuf runtime (C++, Java, Go, Python all
use reference types), and it changes how large or deeply-nested messages behave
under mutation and copying. The second defining trait is fidelity: the project
passes Google's full protobuf conformance suite, so binary and JSON output are
wire-compatible with any other conformant implementation[^1]. That compatibility
is the whole point — the same `.proto` generates Swift here and Java/C++/Go
elsewhere, and the bytes interoperate with no extra work.

Note the naming: the module is `SwiftProtobuf`, the plugin binary is
`protoc-gen-swift`, and neither is a full gRPC stack. RPC lives in the separate
grpc/grpc-swift project, which depends on this one for message types.

## Getting Started

Install both the compiler and the plugin. On macOS with Homebrew this is one step:

```bash
brew install swift-protobuf   # installs protoc + protoc-gen-swift
```

Or build the plugin from source and put it on your `PATH`:

```bash
git clone https://github.com/apple/swift-protobuf.git
cd swift-protobuf
git checkout tags/1.27.0        # pin to a released tag
swift build -c release          # produces .build/release/protoc-gen-swift
```

Add the runtime to `Package.swift`, matching the plugin version:

```swift
dependencies: [
    .package(url: "https://github.com/apple/swift-protobuf.git", from: "1.27.0"),
],
targets: [
    .target(name: "MyTarget",
            dependencies: [.product(name: "SwiftProtobuf", package: "swift-protobuf")]),
]
```

Generate and use:

```bash
protoc --swift_out=. DataModel.proto
```

```swift
var info = BookInfo()
info.id = 1734
info.title = "Really Interesting Book"

let bytes: [UInt8] = try info.serializedBytes()          // compact binary
let decoded = try BookInfo(serializedBytes: bytes)
let json: Data = try info.jsonUTF8Data()                 // canonical protobuf JSON
```

## Architecture / How It Works

The code generator is itself a Swift program that speaks `protoc`'s plugin
protocol: `protoc` parses `.proto` files into a `CodeGeneratorRequest` (a
serialized FileDescriptorSet) and pipes it to `protoc-gen-swift` on stdin;
the plugin emits Swift source on stdout. So the plugin never parses `.proto`
syntax itself — it consumes descriptors, which is why you always need a real
`protoc` alongside it[^3].

Generated messages conform to the `Message` protocol and store fields in plain
Swift properties. Serialization is driven by generated `traverse`/`decodeMessage`
methods that walk fields in tag order rather than by runtime reflection, which
keeps encoding allocation-light. The library exposes a `SwiftProtobufContiguousBytes`
protocol so `serializedBytes()` can resolve to either `Data` or `[UInt8]` at the
call site without copying through an intermediate.

Value semantics ripple through the whole design. Nested messages are stored
inline (or behind a copy-on-write box for recursion), `oneof` becomes a Swift
`enum` with associated values, and proto3 scalar fields default to their zero
value rather than being nullable — presence for scalars requires the `optional`
keyword (proto3 field presence) or a wrapper type. Unknown fields are preserved
by default so round-tripping through an older schema does not drop data.

The generator supports both proto2 and proto3, and offers options like
`swift_prefix` (introduced in protoc 3.2.0), visibility control, and file-naming
strategies (`FullPath`, `PathToUnderscores`, `DropPath`). These are passed as
`--swift_opt` flags and materially change the shape of generated symbols.

## Production Notes

- **Version lockstep is mandatory.** The generated `.pb.swift` code and the
  `SwiftProtobuf` runtime are a matched pair. Generating with one plugin version
  and linking a different runtime major/minor can fail to compile or, worse,
  behave subtly wrong. Pin the plugin tag and the SwiftPM `from:` version to the
  same number, and regenerate when you bump.

- **Toolchain floor moves.** Recent releases require Swift 6.1 / Xcode 16.3+[^2].
  Apps stuck on older Xcode must pin an older SwiftProtobuf tag; there is no
  back-support branch. This bites CI images and enterprise build fleets that lag
  Xcode.

- **Generation is a build-time dependency you must wire in.** Nothing regenerates
  `.pb.swift` automatically. Teams either commit generated files (simple, but
  drift-prone) or add a SwiftPM build-tool plugin / Makefile / script step. The
  SwiftPM plugin path needs `protoc` present in the build environment, which
  complicates hermetic and sandboxed builds.

- **Value semantics have a cost.** Large messages copied by value across many
  boundaries can allocate more than the class-based runtimes. Copy-on-write
  mitigates this, but mutating one field of a big shared message triggers a full
  copy. Profile before assuming protobuf structs are free to pass around.

- **proto3 scalar presence surprises.** A `string` field that is `""` and one that
  was never set are indistinguishable on the wire in plain proto3, and both encode
  to nothing. If you need to tell "empty" from "absent", use `optional` fields or
  wrapper types — this is a protobuf semantics issue, not a Swift bug, but it
  regularly trips up teams new to proto3.

- **JSON is canonical protobuf JSON, not free-form.** The JSON codec follows the
  protobuf JSON mapping (camelCase field names, base64 bytes, string-encoded
  64-bit ints). It is not a drop-in replacement for `Codable` and will not match
  a hand-designed REST payload.

- **No RPC here.** For services you add grpc/grpc-swift on top; this repo only
  handles message serialization.

## When to Use / When Not

**Use when:**
- You already use Protocol Buffers and need Swift clients/servers that interoperate
  byte-for-byte with other languages.
- You want schema-driven, versioned wire formats with forward/backward compatibility.
- You want compact binary serialization that outperforms JSON for size and speed.
- You're building gRPC services in Swift (via grpc-swift, which sits on this).

**Avoid when:**
- You only serialize Swift-to-Swift and never cross a language or version boundary
  — `Codable` is simpler and needs no external toolchain.
- You can't run `protoc` in your build environment or tolerate a codegen step.
- You need a human-editable config/document format — protobuf is a machine wire
  format, not a config language.
- You need a schemaless or dynamically-typed payload.

## Alternatives

- protocolbuffers/protobuf — Google's reference implementation; use its C++/Objective-C
  runtime instead when you need the canonical upstream or ObjC interop rather than Swift value types.
- grpc/grpc-swift — use in addition when you need RPC services, not just message encoding.
- apple/swift-nio — use for the networking layer under gRPC; orthogonal, often paired.
- flatbuffers (google/flatbuffers) — use instead when you need zero-copy access and can trade away protobuf's ecosystem.
- Swift `Codable` (standard library) — use instead for Swift-only JSON/plist with no schema or cross-language requirement.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-09 | Repository opened; tracked early Swift + protobuf codegen[^1]. |
| 1.0.0 | 2017 | First stable release; API compatibility promise begins. |
| 1.x | 2017–2026 | Long, incremental 1.x line: proto2/proto3, JSON codec, conformance, Swift-version tracking. |
| 1.27.0 | 2024 | Referenced as a current pin in project docs[^1]. |

## References

[^1]: Swift Protobuf README — features, conformance, install, and quick start. https://github.com/apple/swift-protobuf/blob/main/README.md
[^2]: README "System Requirements" — Swift 6.1 / Xcode 16.3 or later for current releases. https://github.com/apple/swift-protobuf/blob/main/README.md#system-requirements
[^3]: Protocol Buffers documentation — `protoc` compiler and plugin protocol. https://protobuf.dev/

## Tags

swift, protobuf, protocol-buffers, serialization, code-generation, protoc-plugin, apple, grpc, wire-format, apache-2.0
