# JamesNK/Newtonsoft.Json

> Json.NET — the reflection-based JSON serializer that was .NET's de facto default for a decade, now the compatibility-and-flexibility option next to the built-in System.Text.Json.

[GitHub repo](https://github.com/JamesNK/Newtonsoft.Json) ·
[Official website](https://www.newtonsoft.com/json) ·
[License: MIT](https://github.com/JamesNK/Newtonsoft.Json/blob/master/LICENSE.md)

## Overview

Json.NET (NuGet id `Newtonsoft.Json`) is a JSON framework for .NET written by James Newton-King, first released on CodePlex in the late 2000s and moved to GitHub in 2012[^1]. For roughly a decade it was the JSON library for .NET: ASP.NET Web API and early ASP.NET Core shipped it as the default serializer, and it remains one of the most-downloaded packages on NuGet by a wide margin. At ~11.3k stars and ~3.3k forks the GitHub numbers understate its reach — the package is a transitive dependency of a large fraction of the .NET ecosystem.

The defining tension is its relationship with **System.Text.Json**, which Microsoft shipped built into .NET Core 3.0 (2019) as a faster, lower-allocation, `Span`-based alternative. ASP.NET Core switched its default serializer to System.Text.Json at the same time[^2]. Newtonsoft did not become obsolete: it stayed more feature-complete and more lenient than the built-in option (polymorphic type handling, `DataTable`/`dynamic` support, populate-existing-object, `JsonPath` querying, and generally forgiving parsing). The trade is performance and safety — Newtonsoft leans hard on reflection and has permissive defaults that have been the source of real vulnerabilities.

Maintenance is best described as stable but low-velocity. The `master` branch still receives commits, but tagged releases are infrequent — the `13.0.x` line has spanned several years. In practice you adopt it for compatibility or a specific feature System.Text.Json lacks, not because it is under active feature development.

## Getting Started

```bash
dotnet add package Newtonsoft.Json
```

```csharp
using Newtonsoft.Json;

public record Account(string Email, bool Active, IList<string> Roles);

var account = new Account("tom@example.com", true, new[] { "admin", "user" });

// Object -> JSON
string json = JsonConvert.SerializeObject(account, Formatting.Indented);

// JSON -> object
Account back = JsonConvert.DeserializeObject<Account>(json);
```

LINQ to JSON, for when you do not have (or want) a target type:

```csharp
using Newtonsoft.Json.Linq;

JObject o = JObject.Parse(json);
string email = (string)o["Email"];
JToken firstRole = o["Roles"]!.First!;   // JSONPath: o.SelectToken("Roles[0]")
```

## Architecture / How It Works

There are three layered entry points, and most confusion comes from mixing them:

1. **`JsonConvert`** — static convenience wrappers (`SerializeObject` / `DeserializeObject`). Good for one-liners.
2. **`JsonSerializer`** — the real engine. Configure a `JsonSerializerSettings`, get full control over converters, contract resolution, null handling, and reference tracking. `JsonConvert` is a thin facade over this.
3. **LINQ to JSON** (`JToken` / `JObject` / `JArray` / `JValue`) — an in-memory mutable DOM you can query with indexers or `SelectToken` (a JSONPath dialect). This path does no POCO binding.

Under the hood, serialization is driven by **contracts**. A `DefaultContractResolver` reflects over a type once and produces a cached `JsonContract` describing members, converters, and callbacks; `CamelCasePropertyNamesContractResolver` is the common variant. Streaming is handled by `JsonReader`/`JsonWriter` (with `JsonTextReader`/`JsonTextWriter` for text), so large documents can be processed token-by-token without materializing a DOM.

Customization points, roughly in order of how often they are reached for:

- **Attributes** — `[JsonProperty]`, `[JsonIgnore]`, `[JsonConverter]`, `[JsonObject]`, `[JsonExtensionData]`.
- **`JsonConverter`** — write bespoke read/write logic for a type; the escape hatch for anything the contract model cannot express.
- **`IContractResolver`** — reshape naming and member selection globally.
- **`JsonConvert.DefaultSettings`** — a process-wide settings factory, convenient and a frequent source of "why is serialization behaving differently in tests" surprises.

BSON, JSON Schema, and F# support live in separate packages (`Newtonsoft.Json.Bson`, the commercially-licensed `Newtonsoft.Json.Schema`, etc.) rather than the core library. The core targets .NET Framework, .NET Standard 2.0, and modern .NET, which is a large part of why it remains a universal dependency.

## Production Notes

**`TypeNameHandling` is a remote-code-execution footgun.** Setting `TypeNameHandling.Auto/All/Objects` embeds .NET type names in the JSON and instantiates them on read. Deserializing untrusted input with this enabled is a well-known insecure-deserialization vector. The library emits a warning in docs; treat any non-`None` value on an external boundary as a vulnerability.

**MaxDepth and the stack-overflow CVE.** Older versions had no default depth limit, so deeply nested JSON could crash the process (CVE-2024-21907, a denial-of-service via unbounded recursion). Version 13.0.1 introduced a default `MaxDepth` of 64. If you are pinned to an older release, set `MaxDepth` explicitly — this is the single most important upgrade reason for the library.

**Lenient date parsing bites silently.** By default (`DateParseHandling.DateTime`) Newtonsoft inspects strings during read and converts anything that looks like a date into a `DateTime`, altering values you intended to keep as strings and mangling time zones. For round-trip fidelity set `DateParseHandling.None` and handle dates explicitly, or use `DateTimeOffset`.

**Reuse resolvers and settings.** `DefaultContractResolver` caches reflected contracts internally, but that cache lives on the resolver instance. Constructing a fresh `JsonSerializerSettings` with a new contract resolver per call throws the cache away and can dominate CPU under load. Keep a single shared, immutable settings/resolver instance.

**Performance vs System.Text.Json.** Newtonsoft is materially slower and allocates more than the built-in serializer on typical workloads — it is reflection-heavy and string-oriented rather than `Span<byte>`-oriented. It is also not friendly to trimming or Native AOT, because reflection over arbitrary types defeats the linker. On modern .NET, hot paths and AOT builds should prefer System.Text.Json.

**Keeping it in ASP.NET Core.** Since the default flipped, restoring Newtonsoft behavior in an ASP.NET Core app means adding the `Microsoft.AspNetCore.Mvc.NewtonsoftJson` package and calling `.AddNewtonsoftJson()`. This is the standard move when a payload relies on Newtonsoft-only behavior (e.g. `JsonPatch`, which the framework still implements on top of Newtonsoft).

## When to Use / When Not

**Use when:**
- You need behavior System.Text.Json does not (cleanly) cover: `DataTable`, `dynamic`/`ExpandoObject`, populate-into-existing-instance, flexible reference handling, JSONPath queries.
- You are maintaining an existing codebase already built around its attributes and converters — a rewrite to STJ is rarely worth it.
- You need one serializer that behaves identically across .NET Framework and modern .NET.
- You want the most forgiving parser for messy, third-party JSON.

**Avoid when:**
- Throughput or allocation matters on a hot path — use System.Text.Json.
- You are targeting Native AOT or aggressive trimming.
- The input is untrusted and you were tempted by `TypeNameHandling` — the safer default is STJ.
- You are greenfield on modern .NET with plain DTOs; the built-in serializer is the lower-friction default.

## Alternatives

- dotnet/runtime — System.Text.Json, built into the platform; use it when you are on modern .NET and do not need Newtonsoft's flexibility or leniency.
- Tornhoof/SpanJson — very high-throughput `Span`-based serializer; use when raw speed on well-defined DTOs is the priority.
- neuecc/Utf8Json — fast UTF-8 serializer; use for performance, but note it is effectively unmaintained.
- ServiceStack/ServiceStack.Text — compact, fast text serializer; use inside a ServiceStack stack or where its terser output helps.
- neuecc/MessagePack-CSharp — not JSON at all; use when you control both ends and want a compact, fast binary format instead.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2012-02-18 | Source moved to GitHub; repo created[^1]. |
| 6.0 | 2014 | Widely-adopted era; default serializer in ASP.NET Web API. |
| 12.0 | 2019 | Last major line before System.Text.Json shipped in .NET Core 3.0[^2]. |
| 13.0.1 | 2021 | Assemblies signed by default; default `MaxDepth` of 64 added (DoS hardening)[^3]. |
| 13.0.2 | 2022 | Maintenance / bug-fix release. |
| 13.0.3 | 2023 | Current stable line; targeting and bug fixes[^4]. |

## References

[^1]: Newtonsoft.Json GitHub repository (created 2012-02-18). https://github.com/JamesNK/Newtonsoft.Json
[^2]: .NET docs, "How to migrate from Newtonsoft.Json to System.Text.Json" — background on the .NET Core 3.0 default change. https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/migrate-from-newtonsoft
[^3]: CVE-2024-21907 — improper handling of exceptional conditions (stack overflow via deeply nested JSON) affecting versions before 13.0.1. https://github.com/advisories/GHSA-5crp-9r3c-p9vr
[^4]: Newtonsoft.Json release notes. https://github.com/JamesNK/Newtonsoft.Json/releases

## Tags

csharp, dotnet, json, serialization, deserialization, linq-to-json, nuget, library, data-format
