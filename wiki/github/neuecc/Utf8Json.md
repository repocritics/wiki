# neuecc/Utf8Json

> A C# JSON serializer that reads and writes directly to UTF-8 bytes to skip the UTF-16 string step — now archived and pointing users to community forks and System.Text.Json.

[GitHub repo](https://github.com/neuecc/Utf8Json) ·
[License: MIT](https://github.com/neuecc/Utf8Json/blob/master/LICENSE)

## Overview

Utf8Json is a JSON serializer for .NET written by Yoshifumi Kawai (neuecc), the author of MessagePack for C#, MagicOnion, UniRx, and UniTask[^1]. Its defining idea is that most JSON serializers pay for an intermediate UTF-16 `string` or `TextWriter`: they serialize to a .NET string and then call `Encoding.UTF8.GetBytes`, or wrap a `Stream` in a `StreamWriter`. Utf8Json serializes objects directly into a `byte[]` of UTF-8, and deserializes directly from UTF-8 bytes, so the encode/decode round-trip disappears entirely. For workloads whose transport is already UTF-8 (HTTP bodies, sockets, files), this removes a whole allocation-heavy layer[^2].

Architecturally it is a JSON sibling of MessagePack for C#: the same `IJsonFormatter<T>` + resolver model, the same runtime IL code generation, the same "cache the formatter on a static generic field" dispatch. It targets .NET Framework 4.5 and .NET Standard 2.0, with separate distribution (a `.unitypackage`) for Unity and Xamarin[^2].

The important thing to know in 2026 is that **the repository is archived** (read-only since May 2022). The README's first line directs users to a community fork, and neuecc himself has publicly steered new projects toward the built-in `System.Text.Json`, whose UTF-8-first design covers the same ground Utf8Json pioneered[^3]. Treat this page as documentation of a still-widely-referenced but no longer maintained library.

## Getting Started

```
Install-Package Utf8Json
```

```csharp
using Utf8Json;

public class Person { public int Age { get; set; } public string Name { get; set; } }

var p = new Person { Age = 99, Name = "foobar" };

byte[] bytes = JsonSerializer.Serialize(p);        // object -> UTF-8 byte[]
Person p2 = JsonSerializer.Deserialize<Person>(bytes);
string json = JsonSerializer.ToJsonString(p2);     // object -> UTF-16 string
JsonSerializer.Serialize(stream, p2);              // write straight to a Stream
```

By default every public field and property is serialized using its member name. `[DataMember(Name = ...)]` and `[IgnoreDataMember]` from `System.Runtime.Serialization` control naming and exclusion; resolver selection (see below) controls casing, null handling, private-member access, and `DateTime` format (ISO 8601 by default)[^2].

Resolvers are the configuration surface. `StandardResolver.Default` handles the common case; variants like `StandardResolver.AllowPrivateExcludeNullSnakeCase` compose the toggles you want, and `JsonSerializer.SetDefaultResolver(...)` sets the process-wide default. Official extension packages add support for immutable collections (`Utf8Json.ImmutableCollection`), Unity value types (`Utf8Json.UnityShims`), and an ASP.NET Core MVC `OutputFormatter` (`Utf8Json.AspNetCoreMvcFormatter`) that writes directly to the response `Stream`[^2].

## Architecture / How It Works

Serialization is driven by resolvers that produce `IJsonFormatter<T>` instances. For user types, `DynamicObjectResolver` emits IL at runtime — one specialized formatter per type per option set, so there are no per-call branches for "should I allow private members" or "is this camelCase." Formatters are cached on static generic fields rather than a dictionary, avoiding hash lookups on the hot path[^2].

Three optimizations account for most of the measured speed:

- **Pre-encoded property names.** A generated formatter stores each property name already escaped and concatenated with its surrounding `{`, `:`, and `,`. Writing a member is then a raw byte-block copy plus the value, using length-specialized copy routines (`UnsafeMemory.WriteRawN`) rather than `Buffer.BlockCopy`, which carries overhead for small buffers[^2].
- **Direct number formatting.** Integers are written with an `itoa`-style routine straight into the UTF-8 buffer, skipping `int.ToString()` + encode. Doubles use a port of Google's `double-conversion` library for `dtoa`/`atod`[^2][^4].
- **Automata-based member matching on read.** Instead of decoding each property name to a string and comparing, the deserializer slices the raw bytes and matches 8 characters at a time as a `ulong`, walking a generated automaton. Values are parsed with `atoi`/`atod` directly from bytes[^2].

`JsonWriter` is a `struct` over an underlying `byte[]`; the high-level API pools working buffers and avoids allocating below 64K. The design goal was zero incidental allocation and zero boxing across all platforms, including Unity/IL2CPP.

## Production Notes

**It is archived — that is the headline caveat.** No security patches, no bug fixes, no support for newer target frameworks land. The last published NuGet package predates .NET 5, so on modern runtimes you are running a library frozen at the .NET Standard 2.0 era. Any new project should default to `System.Text.Json`; migrating an existing one is the standing recommendation from the maintainer[^3].

**AOT / IL2CPP needs pre-generation.** The default `DynamicObjectResolver` uses runtime IL emission, which does not exist on AOT targets (iOS, IL2CPP, some Xamarin/console configurations). On those platforms you must pre-generate formatters/resolvers ahead of time or serialization throws at runtime — the same constraint MessagePack for C# has, and a frequent first-deploy surprise for Unity teams[^2].

**Spec strictness is loose in spots.** Utf8Json is UTF-8-only by design, which is compliant with RFC 8259, but it also accepts JSON with comments (single- and multi-line) that stricter parsers reject, and it has historically been lenient about some malformed input rather than throwing. If you rely on it as a validating parser you will be disappointed; validate separately.

**Immutable-constructor selection differs from MessagePack.** For records/immutable types Utf8Json picks the constructor with the *most* matched arguments by name (case-insensitive); MessagePack for C# picks the *least*. The maintainer notes the MessagePack behavior was a design mistake — but if you port code or expectations between the two, this asymmetry bites[^2]. Use `[SerializationConstructor]` to pin the choice.

**Open issues are frozen.** With the repo read-only, the 170-plus open issues will not be addressed upstream; behavioral bugs live on unless a fork you depend on has patched them.

## When to Use / When Not

**Use when:**
- You are maintaining an existing system already built on Utf8Json and a rewrite is not justified — it still works and is fast.
- You are on a legacy .NET Framework / .NET Standard 2.0 target where `System.Text.Json`'s newer features are unavailable and you need UTF-8-direct throughput.

**Avoid when:**
- You are starting anything new — use `System.Text.Json` (built in, maintained, same UTF-8-first philosophy).
- You need current security patches, source-generator AOT support, or modern .NET target frameworks.
- You need strict spec-conformant JSON validation.

## Alternatives

- dotnet/runtime (System.Text.Json) — the built-in, maintained successor with an AOT-friendly source generator; use it for essentially all new .NET code.
- neuecc/MessagePack-CSharp — same author, same formatter/resolver architecture, binary output; use when you control both ends and want smaller/faster than JSON.
- Cysharp/MemoryPack — the author's modern zero-encoding binary serializer for .NET; use when you want his latest work and don't need human-readable output.
- JamesNK/Newtonsoft.Json — the feature-rich, forgiving standard; use when flexibility, LINQ-to-JSON, and broad ecosystem support matter more than raw speed.
- Tornhoof/SpanJson — a similar UTF-8-direct, allocation-light JSON serializer; use as an alternative UTF-8-first library when System.Text.Json doesn't fit.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-09 | First release; UTF-8-direct serializer sharing MessagePack for C#'s architecture[^1]. |
| 1.x | 2018–2019 | Extension packages (ImmutableCollection, UnityShims, AspNetCoreMvcFormatter); last NuGet updates in this line. |
| archived | 2022-05 | Repository set read-only; README points users to a community fork and, in practice, System.Text.Json[^3]. |

## References

[^1]: neuecc (Yoshifumi Kawai), GitHub profile and OSS portfolio (MessagePack-CSharp, MagicOnion, UniRx, UniTask). https://github.com/neuecc
[^2]: Utf8Json README — architecture, resolvers, built-in types, and optimization notes. https://github.com/neuecc/Utf8Json
[^3]: Utf8Json README archive notice ("THIS PROJECT IS ARCHIVED, USE COMMUNITY FORK INSTEAD"); System.Text.Json documentation. https://learn.microsoft.com/en-us/dotnet/api/system.text.json
[^4]: google/double-conversion — the `dtoa`/`atod` implementation Utf8Json ports for double formatting. https://github.com/google/double-conversion

## Tags

csharp, dotnet, json, serialization, serializer, utf-8, performance, unity, archived, high-performance
