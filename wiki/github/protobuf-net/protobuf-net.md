# protobuf-net/protobuf-net

> A contract-based Protocol Buffers serializer for .NET that maps attributed C# types to Google's protobuf wire format.

[GitHub repo](https://github.com/protobuf-net/protobuf-net) ·
[Documentation](https://protobuf-net.github.io/protobuf-net/) ·
[License: Apache-2.0](https://github.com/protobuf-net/protobuf-net/blob/main/Licence.txt)

## Overview

protobuf-net is Marc Gravell's .NET implementation of Google's Protocol Buffers, written and maintained since 2008[^1]. Unlike Google's own C# port, it is *contract-based* rather than *schema-first*: you annotate ordinary C# classes with `[ProtoContract]` / `[ProtoMember(n)]` attributes (or configure a `RuntimeTypeModel` at runtime), and the library serializes them to the protobuf binary format. The API deliberately mirrors `XmlSerializer` / `DataContractSerializer`, which makes it the path of least resistance for .NET developers who want compact binary output without writing `.proto` files first.

The defining tension is *idiomatic .NET vs. cross-language interoperability*. protobuf-net produces valid protobuf wire bytes, but its handling of .NET-only concepts — inheritance (`[ProtoInclude]`), `DateTime`, `TimeSpan`, `decimal`, and `Guid` — relies on protobuf-net-specific "bcl" message shapes that other languages' protobuf libraries do not understand. Stay within `int`, `string`, `bytes`, nested contracts, and repeated fields and the output is fully portable; reach for the .NET conveniences and you have quietly created a .NET-only format that happens to be protobuf-encoded.

The project originated on Google Code (SVN) and migrated to GitHub in 2014[^2]. It remains one of the most-used serialization libraries in the .NET ecosystem, with the `protobuf-net` NuGet package having hundreds of millions of downloads across its history.

## Getting Started

```ps
Install-Package protobuf-net
```

Decorate types with contract attributes — field numbers, not names, are what gets written:

```csharp
using ProtoBuf;

[ProtoContract]
class Person {
    [ProtoMember(1)] public int Id { get; set; }
    [ProtoMember(2)] public string Name { get; set; }
    [ProtoMember(3)] public Address Address { get; set; }
}

[ProtoContract]
class Address {
    [ProtoMember(1)] public string Line1 { get; set; }
    [ProtoMember(2)] public string Line2 { get; set; }
}
```

```csharp
var person = new Person {
    Id = 12345, Name = "Fred",
    Address = new Address { Line1 = "Flat 1", Line2 = "The Meadows" }
};

using (var file = File.Create("person.bin"))
    Serializer.Serialize(file, person);

Person copy;
using (var file = File.OpenRead("person.bin"))
    copy = Serializer.Deserialize<Person>(file);
```

Field numbers should be positive, ideally `<= 536870911`, and must avoid the reserved `19000-19999` range. The number — not the member name — is the on-wire identity: you can rename a property freely, but changing its `[ProtoMember]` number changes the data[^3].

## Architecture / How It Works

At the center is `RuntimeTypeModel`. `Serializer.Serialize/Deserialize` are thin shortcuts over `RuntimeTypeModel.Default`; anything the attributes express can equally be configured imperatively on a model at startup (`RuntimeTypeModel.Default.Add(typeof(T), ...)`), which is how you serialize types you cannot annotate.

For speed, protobuf-net does not reflect per-call. The first time a type is used, it builds a specialized serializer and — on platforms that permit it — emits IL via `Reflection.Emit` so subsequent calls run near hand-written code. A model can be frozen and compiled ahead of time (`model.Compile()` / `CompileInPlace()`), or emitted to a standalone serialization assembly.

Wire mapping details worth knowing:

- **Fields are optional and defaults are omitted.** protobuf has no concept of "present but default"; a zero/`null`/empty value is simply not written. To distinguish "unset" from "default" you must use nullable types or track presence yourself.
- **Inheritance is explicit and non-standard.** `[ProtoInclude(n, typeof(Derived))]` encodes the subtype as a nested field under integer key `n`. No other protobuf implementation reads this convention.
- **BCL types use wrapper messages.** `DateTime`, `TimeSpan`, `decimal`, and `Guid` serialize via protobuf-net's `bcl.proto` definitions rather than well-known types like `google.protobuf.Timestamp`.
- **Surrogates** (`model.SetSurrogate`) let an unserializable type be swapped for a serializable stand-in at the model layer.

Two schema directions are supported: annotate C# and (optionally) emit a `.proto`, or start from a `.proto` and generate C# with **protogen** (available as the `protobuf-net.Protogen` global tool or the online tool at protogen.marcgravell.com). The `protobuf-net.Grpc` sibling project builds code-first gRPC services on the same model.

## Production Notes

**AOT / IL2CPP / NativeAOT is the number-one footgun.** The runtime IL-emit fast path requires `Reflection.Emit`, which is unavailable on iOS, Unity IL2CPP, Xamarin AOT, and .NET NativeAOT. On those targets an un-prepared model throws at runtime rather than at build time. Mitigations: precompile the model (`CompileInPlace()` on startup), generate a serialization assembly ahead of time, or add `protobuf-net.BuildTools`[^4], which contributes Roslyn analyzers/source generation to catch contract mistakes at compile time. Budget real time for this before shipping to a constrained runtime.

**Cross-language interop is opt-in, not automatic.** If any consumer is not itself protobuf-net (a Go, Java, Python, or Google.Protobuf peer), audit every contract for inheritance, `DateTime`/`decimal`/`Guid`, and grouped encoding. Emit the `.proto` protobuf-net generates and diff it against what the other side expects. Teams routinely discover incompatibility only after the first non-.NET consumer appears.

**Model configuration is startup-time.** A `RuntimeTypeModel` must have its types added before first use; adding or mutating types after the model has been compiled/frozen throws. Register everything during application startup, once, and treat the default model as effectively immutable thereafter. The serialize/deserialize paths themselves are thread-safe.

**v2 → v3 is a breaking upgrade.** Version 3 was a substantial rewrite that split the code into `protobuf-net.Core`, changed several default behaviors, tightened nullable/`proto3` semantics, and dropped some legacy features and older target frameworks[^5]. Do not treat a v3 bump as a routine update — re-serialize/re-read a representative payload corpus and read the v3 release notes before upgrading a system with persisted data on disk.

**Schema evolution discipline.** Because field numbers are the contract, the usual protobuf rules apply: never reuse or renumber a retired field, add new fields with new numbers, and keep old numbers reserved. Renaming members is safe; renumbering silently corrupts old data.

## When to Use / When Not

**Use when:**
- You want compact binary serialization for .NET-to-.NET communication or storage and prefer annotating C# over authoring `.proto`.
- You are migrating from `XmlSerializer` / `DataContractSerializer` and want a familiar, attribute-driven model.
- You need code-first gRPC in .NET (via `protobuf-net.Grpc`).
- You already have `.proto` schemas but want idiomatic generated C# and a runtime that feels like .NET.

**Avoid when:**
- Cross-language interoperability with non-.NET services is a hard requirement — the official Google.Protobuf port is safer because its type mapping is standard by construction.
- You target AOT/IL2CPP/NativeAOT and cannot invest in precompilation or the build-time tooling.
- You want raw throughput for an all-.NET system with no protobuf-format constraint — MessagePack-CSharp and MemoryPack typically benchmark faster.
- Human-readable or JSON-interop payloads are needed — reach for `System.Text.Json`.

## Alternatives

- protocolbuffers/protobuf — Google's official C# port (`Google.Protobuf`); schema-first codegen with guaranteed cross-language wire compatibility. Use when interop with non-.NET peers is non-negotiable.
- neuecc/MessagePack-CSharp — MessagePack (not protobuf) binary format, very fast, strong Unity story. Use when you control both ends and want raw speed over protobuf compatibility.
- Cysharp/MemoryPack — zero-encoding .NET-only serializer optimized for throughput. Use for internal .NET services where no external format contract exists.
- microsoft/bond — cross-platform schema-based serialization with its own IDL. Use when you want a schema-first system outside the protobuf format.
- protobuf-net/protobuf-net.Grpc — same family, code-first gRPC on top of this model. Use when the goal is gRPC services rather than bare serialization.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2008 | First release by Marc Gravell on Google Code (SVN)[^1]. |
| v2 | ~2011 | `RuntimeTypeModel`, precompiled serializers, imperative configuration. |
| — | 2014-06 | Source migrated from Google Code to GitHub[^2]. |
| v3.0 | 2020 | Major rewrite; `protobuf-net.Core` split, changed defaults, dropped legacy targets and frameworks[^5]. |

Supported runtimes as of the current line: .NET 6.0+, .NET Standard 2.0/2.1, and .NET Framework 4.6.2+.

## References

[^1]: protobuf-net Licence.txt — ".NET implementation is Copyright 2008 onwards Marc Gravell". https://github.com/protobuf-net/protobuf-net/blob/main/Licence.txt
[^2]: GitHub repository metadata, `created_at` 2014-06-16 (project predates this on Google Code). https://github.com/protobuf-net/protobuf-net
[^3]: protobuf-net README, "Notes for Identifiers". https://github.com/protobuf-net/protobuf-net
[^4]: protobuf-net.BuildTools documentation. https://protobuf-net.github.io/protobuf-net/build_tools
[^5]: protobuf-net v3 release notes. https://protobuf-net.github.io/protobuf-net/3_0

## Tags

csharp, dotnet, serialization, protocol-buffers, protobuf, binary-format, grpc, contract-serializer, aot, marc-gravell
