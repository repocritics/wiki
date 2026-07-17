# Cysharp/MemoryPack

> Zero-encoding binary serializer for C# and Unity that copies memory directly instead of encoding a portable wire format.

[GitHub repo](https://github.com/Cysharp/MemoryPack) ·
[License: MIT](https://github.com/Cysharp/MemoryPack/blob/main/LICENSE)

## Overview

MemoryPack is a binary serializer for .NET and Unity built by Yoshifumi Kawai (neuecc) at Cysharp. It is his fourth serializer, following ZeroFormatter, Utf8Json, and MessagePack for C#[^1]. The central design idea is "zero encoding": rather than run VarInt encoding, tag writing, and field-name emission the way most binary formats do, MemoryPack copies as much of the in-memory C# representation as possible directly into the output buffer. For arrays of unmanaged structs this collapses to a single `memcpy`, which is where its headline "x50–x200 faster than other serializers" figures for struct arrays come from[^1].

The consequence — and the defining tension — is that MemoryPack's wire format is deliberately *not* a neutral interchange format. It mirrors C# memory layout and type ordering, so both ends of the wire must share the same type definitions compiled the same way. This is the opposite of Protobuf or MessagePack, which trade some speed for a self-describing, cross-language, long-lived schema. MemoryPack is the right tool when both peers are .NET/Unity code you control (game client↔server, Redis cache, IPC, save files); it is the wrong tool when you need polyglot interop or a stable format that outlives your type definitions.

As of this writing the repository has 4,629 stars and 307 forks, with commits within the last week — an actively maintained project with steady adoption inside the .NET and Unity performance niche rather than mainstream ubiquity[^2].

## Getting Started

```
PM> Install-Package MemoryPack
```

Best performance targets .NET 7+; the minimum is .NET Standard 2.1. The source generator requires Roslyn 4.3.1 (Visual Studio 2022 17.3 / .NET SDK 6.0.401 or newer)[^1].

```csharp
using MemoryPack;

[MemoryPackable]                 // triggers the source generator
public partial class Person      // `partial` is mandatory
{
    public int Age { get; set; }
    public string Name { get; set; }
}

var v   = new Person { Age = 40, Name = "John" };
byte[] bin = MemoryPackSerializer.Serialize(v);
var val = MemoryPackSerializer.Deserialize<Person>(bin);
```

`Serialize` also targets `IBufferWriter<byte>` and `Stream`; `Deserialize` accepts `ReadOnlySpan<byte>`, `ReadOnlySequence<byte>`, and `Stream`. The buffer-writer overload is the fast path — the `byte[]` return is simpler and nearly as fast for cases like `RedisValue`[^1].

## Architecture / How It Works

MemoryPack generates code at build time via an **Incremental Source Generator** — no `IL.Emit`, no reflection-based dynamic codegen — which is what makes it Native AOT friendly and Unity IL2CPP compatible[^1]. Annotating a type with `[MemoryPackable] partial` causes the generator to emit an `IMemoryPackable<T>` implementation you can inspect (`*.MemoryPackFormatter.g.cs`). There are 35 diagnostic rules (`MEMPACK001`–`MEMPACK035`) that surface serialization mistakes as compile errors rather than runtime failures.

The format carries no member names. Members are written **in declaration order** (parent → child for inheritance), and deserialization reads them back in that same positional order. A struct or record struct that contains only unmanaged fields is treated as a raw memory blob and serialized/deserialized straight from memory with no per-member logic. Reference-typed classes serialize public instance properties and fields by default; `[MemoryPackIgnore]` / `[MemoryPackInclude]` adjust the set, and `[MemoryPackable(SerializeLayout.Explicit)]` with `[MemoryPackOrder]` pins an explicit order.

Beyond the core path it supports polymorphism via `[MemoryPackUnion]` (tag→type mapping, tags 0–65535, cheapest under 250), circular references, `[MemoryPackConstructor]` selection with case-insensitive parameter-name matching, serialization callbacks (`OnSerializing`/`OnSerialized`/`OnDeserializing`/`OnDeserialized`, including `ref writer/reader` variants for custom headers), deserialize-into-existing-instance, `PipeWriter`/`PipeReader` streaming, TypeScript codegen, and an ASP.NET Core formatter. String encoding is selectable between UTF-8 (default, smaller for ASCII) and UTF-16 (faster, matches C#'s internal representation); the encoding is auto-detected on read.

## Production Notes

- **The layout *is* the schema.** Because members serialize positionally, reordering, inserting, or removing a member silently changes the binary layout. For unmanaged structs there is no per-field marker at all, so a layout mismatch does not throw — it deserializes garbage. Treat any change to a serialized type as a wire-breaking change unless you are appending under version-tolerant rules.
- **Version tolerance is opt-in and tiered.** The fast/default mode tolerates appending new members at the end only. Full version tolerance (reordering/removal safety) exists but adds per-member length prefixes and is not the default — decide the mode before you ship data you must read back later.
- **Endianness.** The zero-copy design writes native-endian bytes for unmanaged data. Payloads produced on the near-universal little-endian hardware are not portable to big-endian platforms; do not assume the format is architecture-neutral.
- **Not an interchange format.** Despite the TypeScript generator and ASP.NET formatter, this is not a substitute for Protobuf/MessagePack when a non-.NET consumer, a public API, or a decade-long-stable format is involved. The TS codegen covers client/server sharing of *your* types, not general polyglot use.
- **Untrusted input.** As with any binary deserializer, do not deserialize attacker-controlled bytes into rich object graphs without bounds. Union deserialization instantiates registered types by tag; keep the union set closed and validate lengths.
- **Unity specifics.** `ModuleInitializer` auto-registration is unavailable in Unity, so union/external formatters must be registered manually at startup (e.g. `YourUnionFormatterInitializer.RegisterFormatter()`). Unity support requires 2021.3+ and IL2CPP is handled via the .NET source generator path[^1].
- **Toolchain floor.** Old Roslyn/IDE versions silently degrade the generator experience; MEMPACK diagnostics and generated-code navigation assume VS 2022 17.3+ or an equivalent SDK.

## When to Use / When Not

**Use when:**
- Both peers are .NET/Unity code you compile and version together (game netcode, IPC, Redis/cache payloads, local save files).
- Throughput/allocations dominate and you are serializing large arrays of unmanaged structs.
- You need Native AOT or Unity IL2CPP with no runtime IL generation.

**Avoid when:**
- A non-.NET system must read the bytes, or the format is a public/long-lived contract.
- Schemas evolve frequently and independently on each side without coordinated deploys.
- You want a self-describing, debuggable, or human-readable payload.

## Alternatives

- neuecc/MessagePack-CSharp — same author; use it when you need a cross-language, self-describing format and can spend some speed.
- protobuf-net/protobuf-net — use it for schema-first, polyglot, long-term-stable wire contracts.
- dotnet/orleans (Orleans.Serialization) — use it when you want a version-tolerant serializer designed around evolving distributed types.
- google/flatbuffers — use it for zero-copy random access across languages without full deserialization.
- dotnet/runtime (System.Text.Json) — use it when interop, readability, and debuggability outweigh raw throughput.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repository created | 2022-09-03 | Public repo opened at Cysharp[^2]. |
| Public 1.x release | 2022 | Announced as neuecc's 4th serializer; .NET 7 / C# 11 / Incremental Source Generator design[^1]. |
| Ongoing | through 2026-07 | Actively maintained; MIT-licensed; last push 2026-07-08[^2]. |

## References

[^1]: MemoryPack README, Cysharp/MemoryPack. https://github.com/Cysharp/MemoryPack
[^2]: GitHub REST API, repos/Cysharp/MemoryPack (metadata fetched 2026-07). https://api.github.com/repos/Cysharp/MemoryPack

## Tags

csharp, dotnet, unity, serialization, binary-serializer, zero-copy, source-generator, native-aot, performance, il2cpp
