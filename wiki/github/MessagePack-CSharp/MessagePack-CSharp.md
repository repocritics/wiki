# MessagePack-CSharp/MessagePack-CSharp

> A fast binary serializer for C# implementing the MessagePack format, with attribute-driven contracts, pluggable resolvers, and AOT support for Unity and NativeAOT.

[GitHub repo](https://github.com/MessagePack-CSharp/MessagePack-CSharp) ·
[msgpack.org](https://msgpack.org/) ·
[License: MIT](https://github.com/MessagePack-CSharp/MessagePack-CSharp/blob/master/LICENSE)

## Overview

MessagePack for C# is a serializer that encodes .NET objects into the [MessagePack](https://msgpack.org/) binary format — a compact, schema-less, self-describing wire format positioned between JSON (human-readable, verbose) and Protocol Buffers (schema-required, cross-language). It was written by Yoshifumi Kawai (neuecc) starting in 2017[^1] and is now a [.NET Foundation](https://dotnetfoundation.org/) project, with Andrew Arnott (AArnott) as the primary long-term maintainer. As of 2026 it sits at roughly 6.7k stars, is actively maintained (v3.1.7 shipped June 2026, commits within the same week), and is a de-facto default binary serializer in the .NET game and RPC ecosystems.

Its defining property is codegen-driven speed. Rather than reflecting over types at serialize time, it generates per-type formatter code — historically via runtime IL emission (`Reflection.Emit`), and since v3 also via Roslyn source generators for ahead-of-time targets. The README's headline claim is "10x faster than MsgPack-Cli"[^2]; independent of the exact multiplier, it is consistently among the fastest general-purpose .NET serializers, at the cost of a contract model you must opt into and understand.

The central tension is that same contract model. MessagePack for C# is fast and compact precisely because it prefers integer-indexed keys serialized as positional arrays — which makes the wire format terse but fragile across schema changes, and opaque when debugging. You trade JSON's forgiving, name-based flexibility for a format where field order and index stability are load-bearing. The typeless / `BinaryFormatter`-style mode, which removes that discipline, is also where the sharpest security footguns live.

## Getting Started

```ps1
Install-Package MessagePack
```

Annotate types with `[MessagePackObject]` and members with `[Key]`, then call the static serializer:

```csharp
using MessagePack;

[MessagePackObject]
public class Person
{
    [Key(0)] public int Age { get; set; }
    [Key(1)] public string FirstName { get; set; }
    [Key(2)] public string LastName { get; set; }

    [IgnoreMember]
    public string FullName => FirstName + LastName;
}

var p = new Person { Age = 99, FirstName = "hoge", LastName = "huga" };

byte[] bytes = MessagePackSerializer.Serialize(p);          // -> [99,"hoge","huga"]
Person back = MessagePackSerializer.Deserialize<Person>(bytes);

// Inspect any blob as JSON for debugging:
string json = MessagePackSerializer.ConvertToJson(bytes);
```

Integer keys serialize to a MessagePack array (compact, fast, field names lost). String keys — or `[MessagePackObject(keyAsPropertyName: true)]` — serialize to a map (larger, self-describing). The `ContractlessStandardResolver` lets you skip attributes entirely and serialize plain POCOs à la Json.NET, at the cost of always using string-keyed maps.

## Architecture / How It Works

Three layers stack from convenient to primitive:

1. **High-level API** — `MessagePackSerializer.Serialize<T>/Deserialize<T>`, the entry point most code uses.
2. **Formatters** — `IMessagePackFormatter<T>` implements read/write for one type. Built-ins cover primitives, most BCL collections, tuples, immutable collections, `DateTime`, `Guid`, `System.Numerics` types, and more.
3. **Reader/Writer primitives** — `MessagePackReader` and `MessagePackWriter` are `ref struct`s that operate over `ReadOnlySequence<byte>` and `IBufferWriter<byte>`, giving allocation-light, `Span`-based access to the raw format.

The extension point that ties formatters together is `IFormatterResolver`. A resolver maps a runtime type to the formatter that handles it; `StandardResolver` composes the built-ins. This is where MessagePack's per-type code generation happens. Under v1/v2 the `DynamicObjectResolver` family emits IL at first use via `Reflection.Emit` — fast at steady state, but requires a JIT and therefore fails on AOT-only runtimes. Since **v3 (December 2024)** a Roslyn source generator produces resolver/formatter code at compile time[^3], which is what makes NativeAOT and Unity IL2CPP work without a runtime codegen step.

The **v2 rewrite (December 2019)** was the architecturally significant one: it re-based the whole library on `System.Buffers` primitives (`ReadOnlySequence<byte>`, `IBufferWriter<byte>`, the `ref struct` reader/writer), aligning it with `System.IO.Pipelines` and eliminating intermediate byte-array copies[^4]. Code written against the v1 `MessagePackSerializer.Serialize(obj)` shape largely still compiles, but the low-level API changed completely.

Compression is built in: LZ4 in two modes — `Lz4Block` (whole payload) and `Lz4BlockArray` (chunked, streaming-friendly) — selected via `MessagePackSerializerOptions`. It is a format-internal extension type, so a compressed blob is still a valid MessagePack extension and round-trips through the same API.

## Production Notes

**The typeless / `BinaryFormatter` path is a remote-code-execution class of risk.** `TypelessFormatter` and the `Typeless` resolvers embed .NET type names in the payload and instantiate whatever type the incoming data names. Deserializing untrusted input in this mode is the same vulnerability family that got `BinaryFormatter` deprecated. Do not use typeless serialization on data crossing a trust boundary.

**Untrusted input needs `MessagePackSecurity.UntrustedData`.** By default the deserializer is tuned for trusted peers. Hostile payloads can trigger hash-collision DoS on dictionary/hash-set types and stack overflow via deeply nested structures. Pass `options.WithSecurity(MessagePackSecurity.UntrustedData)` for any externally-sourced bytes; it randomizes hash seeds and caps nesting depth.

**AOT requires setup you cannot skip.** On Unity IL2CPP and NativeAOT there is no runtime IL emission, so the dynamic resolvers throw. You must generate a resolver ahead of time — historically the `mpc` command-line tool, now the v3 source generator — and register it before first use. Forgetting this produces runtime "formatter not found" / `TypeInitializationException` failures that only appear on device, not in the editor.

**Schema evolution is manual and unforgiving with integer keys.** Missing keys deserialize to `default` (reference types become `null`) with no error, so a version mismatch can surface as silent data loss rather than an exception. Never reuse a retired integer key. Large gaps in the index sequence insert `null` placeholders into the array and bloat the payload. Teams that need forgiving cross-version evolution often prefer string keys despite the size cost.

**`init` setters on generic classes hit a CLR bug.** With the public-only `DynamicObjectResolver`/`StandardResolver`, a CLR limitation prevents the most efficient generated code from invoking `init` setters in generic types[^5]; use the `*AllowPrivate` resolvers or avoid `init` there.

**`MessagePackSerializer.DefaultOptions` is process-global mutable state.** Setting it (e.g. to swap in the contractless or a compression-enabled resolver) is convenient in an app entry point but a hazard in library code, where it silently changes behavior for every other consumer in the process.

## When to Use / When Not

**Use when:**
- You control both serialize and deserialize ends (RPC between your own services, game client/server, caches) and want small, fast payloads.
- You are on Unity or NativeAOT and need a serializer with a real AOT story.
- You want binary MessagePack interop with other languages while keeping idiomatic C# contracts.

**Avoid when:**
- You need human-readable output or loose, name-based schema tolerance — reach for `System.Text.Json`.
- You must deserialize untrusted, self-describing payloads with arbitrary types — the typeless mode's risk outweighs its convenience.
- You need a cross-language, schema-first contract with generated stubs and strong forward/backward compatibility guarantees — Protocol Buffers fits better.
- Your data is .NET-only and you want maximum speed with less contract ceremony — MemoryPack (same author) may serialize faster.

## Alternatives

- protocolbuffers/protobuf — schema-first (`.proto`) cross-language format with generated code and disciplined versioning; use when interop and long-term schema compatibility matter more than raw C# speed.
- protobuf-net/protobuf-net — attribute-driven Protocol Buffers for .NET, a closer ergonomic analogue; use when you want protobuf on the wire but MessagePack-style C# annotations.
- Cysharp/MemoryPack — the same author's newer zero-encoding, .NET-only serializer; use when both ends are .NET and you want the fastest option and don't need cross-language interop.
- dotnet/runtime (System.Text.Json) — the built-in JSON serializer; use when readability, debuggability, and web interop beat binary size and speed.
- msgpack/msgpack-cli — the older .NET MessagePack library this project set out to outperform; effectively legacy, prefer this project for new work.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2017 | Initial release by neuecc; runtime IL codegen resolvers, indexed/string keys, LZ4[^1]. |
| 2.0.323 | 2019-12-16 | Rewrite on `System.Buffers`; `ref struct` reader/writer over `ReadOnlySequence<byte>`; .NET Foundation project[^4]. |
| 2.1.x | 2020-01 | Post-rewrite hardening; broadened built-in type and platform support. |
| 3.0.x | 2024-12-06 | Roslyn source-generator resolvers for AOT (NativeAOT, IL2CPP); reduced reliance on runtime emission[^3]. |
| 3.1.7 | 2026-06-09 | Latest release as of this writing; continued .NET 8+ and Unity maintenance. |

## References

[^1]: Repository created 2017-02-13; original author Yoshifumi Kawai (neuecc). https://github.com/MessagePack-CSharp/MessagePack-CSharp
[^2]: Project README performance claim ("10x faster than MsgPack-Cli"). https://github.com/MessagePack-CSharp/MessagePack-CSharp#performance
[^3]: v3.0.3 tag dated 2024-12-06 (git tag commit date); source-generator-based AOT resolver generation. https://github.com/MessagePack-CSharp/MessagePack-CSharp/releases
[^4]: v2.0.323 tag dated 2019-12-16 (git tag commit date); the `System.Buffers`-based rewrite and low-level API. https://github.com/MessagePack-CSharp/MessagePack-CSharp/blob/master/doc/migration.md
[^5]: README note on the CLR bug affecting `init` setters in generic classes with public-only resolvers. https://github.com/neuecc/MessagePack-CSharp/issues/1134

## Tags

csharp, dotnet, serialization, messagepack, binary-format, lz4, unity, aot, rpc, performance, msgpack, dotnet-foundation
