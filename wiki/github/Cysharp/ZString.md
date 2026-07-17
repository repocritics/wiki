# Cysharp/ZString

> Zero-allocation StringBuilder and String.Format replacement for .NET and Unity, built on structs, `ArrayPool`, and generic append methods.

[GitHub repo](https://github.com/Cysharp/ZString) ·
[NuGet package](https://www.nuget.org/packages/ZString) ·
[License: MIT](https://github.com/Cysharp/ZString/blob/master/LICENSE)

## Overview

ZString is a string-building library from Cysharp — the studio behind UniTask, MessagePack-CSharp, and ZLogger — first released in early 2020[^1]. Its single goal is to eliminate the intermediate allocations that ordinary .NET string construction incurs: `"x:" + x + " y:" + y` is compiled to `String.Concat` over an array of `.ToString()` results, and `string.Format` boxes every value-type argument into `object`. ZString replaces both paths with a struct-based builder that writes each value directly into a pooled buffer, so the only allocation left is the final `string` — and even that can be skipped if the consumer accepts a `Span`.

The library is aimed at hot paths where string churn dominates GC pressure: game loops, high-frequency logging, serializers, and any per-frame UI text update. Its most-cited integration is Unity's TextMeshPro, where `SetText`/`SetTextFormat` write ZString's inner buffer straight into the mesh, achieving genuinely zero-allocation label updates. On .NET Core it targets both UTF-16 (`Span<char>`) and UTF-8 (`Span<byte>`) output directly, so UTF-8 producers (HTTP responses, `System.Text.Json`) avoid a transcoding step.

The defining tension is that ZString buys its performance with mutable structs and pooled buffers, which push memory-management concerns back onto the caller. `Utf16ValueStringBuilder` is a mutable `struct` that must be disposed to return its 64 KB rented buffer, must be passed by `ref`, and — in its fastest `notNested:true` mode — uses a thread-static buffer that cannot be nested. This is the opposite of `System.Text.StringBuilder`'s forgiving reference-type ergonomics, and misuse produces silent buffer corruption or leaked pool arrays rather than exceptions.

## Getting Started

```
# .NET Core / .NET 5+
dotnet add package ZString
```

For Unity, install via UPM git URL (`https://github.com/Cysharp/ZString.git?path=src/ZString.Unity/Assets/Scripts/ZString`) or the `.unitypackage` from the releases page. Minimum supported Unity is 2021.3[^2].

```csharp
using Cysharp.Text;

// Drop-in replacements — allocate only the final string
string a = ZString.Concat("x:", x, " y:", y, " z:", z);
string b = ZString.Format("x:{0}, y:{1:000}, z:{2:P}", x, y, z);
string c = ZString.Join(',', x, y, z);

// Builder — must be disposed to return the pooled buffer
using (var sb = ZString.CreateStringBuilder())
{
    sb.Append("foo");
    sb.AppendLine(42);
    sb.AppendFormat("{0} {1:.###}", "bar", 123.456789);
    string result = sb.ToString();
}

// Unity: write straight into TextMeshPro, no string allocated at all
tmp.SetTextFormat("Position: {0}, {1}, {2}", x, y, z);
```

## Architecture / How It Works

The core is two builder structs, `Utf16ValueStringBuilder` and `Utf8ValueStringBuilder`, each implementing `IBufferWriter<T>` and `IDisposable`. `ZString.CreateStringBuilder()` rents a 64 KB buffer from `ArrayPool<T>.Shared`; `Dispose()` returns it. The one-arg overload `CreateStringBuilder(notNested: true)` instead borrows a `[ThreadStatic]` buffer — faster because it skips the pool, but it is a single shared slot per thread, so any nested builder or intervening `ZString.Concat/Format/Join` call (which also uses the thread-static buffer) corrupts it. The library documents this constraint but cannot enforce it at compile time.

Every append is generic: `Append<T>(T value)` and `AppendFormat<T1..T16>(string, T1..T16)`. The generic signatures are the mechanism that avoids boxing — a `struct` argument stays a `struct` all the way to the formatter instead of being coerced to `object`. ZString ships built-in `TryFormat` implementations for the primitive value types (`Int32`, `Double`, `DateTime`, `Guid`, `Decimal`, and so on) that write digits directly into the buffer; on .NET Standard 2.0 and Unity these number-conversion routines are vendored from `dotnet/runtime`[^3]. Any type without a built-in formatter falls back to `.ToString()` and a copy — so custom structs still allocate unless you register a formatter via `RegisterTryFormat`.

The `AppendFormat`/`Concat`/`Format` API is duplicated across 16 arity overloads (`T1` through `T16`) to keep the boxing-free guarantee for up to sixteen heterogeneous arguments. `PrepareUtf16`/`PrepareUtf8` pre-parse a format template once into a reusable object (analogous to a compiled regex), amortizing template parsing across repeated formats. Because both builders implement `IBufferWriter`, they can be handed to `Utf8JsonWriter` or any serializer that writes to a buffer writer — though doing so requires boxing the mutable struct to the interface, which is itself a footgun (see below).

## Production Notes

- **Disposal is mandatory, not optional.** A builder that is not disposed leaks its 64 KB pooled buffer, and the pool will allocate a fresh 64 KB array for the next caller. In tight loops this reintroduces exactly the allocation ZString exists to remove. Always use `using`, or `try/finally` when passing by `ref`.
- **Mutable-struct copy hazard.** `Utf16ValueStringBuilder`/`Utf8ValueStringBuilder` are mutable structs. Passing one to a method by value copies it; appends to the copy are lost or, worse, the two copies fight over the same underlying buffer state. Helper methods must take `ref Utf16ValueStringBuilder`. When you use `ref`, you cannot use `using` — dispose in a `finally` instead.
- **Boxing to `IBufferWriter` defeats the point unless handled carefully.** Casting the struct to `IBufferWriter<byte>` (e.g. to feed `Utf8JsonWriter`) boxes it; you must unbox the same boxed reference back to the struct to dispose it, or you dispose a copy and leak the original's buffer.
- **`notNested: true` is a loaded gun.** The thread-static fast path conflicts with any nested ZString usage on the same thread, including the static `ZString.Concat/Format/Join` helpers. Safe only when the builder returns its buffer before any other ZString call runs. Reach for it last, after measuring.
- **UTF-8 format strings differ from standard format strings.** The UTF-8 path uses `Utf8Formatter.TryFormat` / `StandardFormat`, whose format symbols are a restricted set (e.g. `D`, `N`, `X`, `G` with an optional precision like `D2`) and do **not** match `String.Format` custom patterns. `DateTime`/`TimeSpan` support is especially limited; complex temporal formatting requires deconstructing into components. Code ported from `string.Format` to `Utf8Format` can silently format differently.
- **Not thread-safe.** A given builder instance is single-threaded, and the thread-static buffer is per-thread by design. Do not share a builder across threads.
- **Wins are real but workload-dependent.** The allocation savings are unambiguous; raw throughput vs. `StringBuilder` is closest to a wash for tiny strings and grows with argument count and format complexity. Profile the actual GC pressure before adopting it framework-wide — for non-hot paths the ergonomic cost outweighs the benefit.

## When to Use / When Not

**Use when:**
- You are on a GC-sensitive hot path — game update loops, per-frame UI text, high-rate logging or serialization.
- You target Unity + TextMeshPro and want zero-allocation label updates.
- You produce UTF-8 output directly and want to skip `char`→`byte` transcoding.
- You have profiled string construction as a measurable source of Gen0 pressure.

**Avoid when:**
- The code is not hot: ordinary `StringBuilder`/interpolation is safer and the allocations are irrelevant.
- You cannot guarantee disciplined `using`/`ref`/dispose usage across the team — misuse leaks pooled buffers.
- Your arguments are mostly custom reference types without registered formatters (they allocate via `ToString()` anyway).
- You want a forgiving, reference-type builder you can freely pass around and store.

## Alternatives

- dotnet/runtime `System.Text.StringBuilder` — the standard, reference-type builder; use when allocation is not the bottleneck and ergonomics matter more.
- C# interpolated string handlers (`DefaultInterpolatedStringHandler`, C# 10+) — the runtime's own low-allocation `$"..."` path; use when you are on a modern TFM and want boxing-free interpolation without a dependency.
- Cysharp/ZLogger — same author's zero-allocation logger built on ZString; use when the string-building is specifically for log output.
- dotnet/runtime `string.Create` + `Span<char>` — hand-rolled span writing; use when you want full control and no dependency for one narrow format.
- Utf8 string literals (`"..."u8`, C# 11+) — for static UTF-8 constants; use when the bytes are known at compile time rather than formatted at runtime.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-02-03 | Cysharp opens the ZString repository[^4]. |
| 1.x | 2020 | Initial releases: struct builders, `T1..T16` Concat/Format, UTF-8 builder, TextMeshPro support[^1]. |
| 2.x | 2021–2024 | `PrepareUtf16`/`PrepareUtf8`, `ZStringWriter : TextWriter`, `IBufferWriter` integration; Unity minimum raised to 2021.3[^2]. |

Exact per-minor release dates are on the GitHub releases page; only the repository-creation date and broad milestone grouping are asserted here to avoid misstating version timing.

## References

[^1]: Yoshifumi Kawai (neuecc), "ZString — Zero Allocation StringBuilder for .NET Core and Unity." https://medium.com/@neuecc/zstring-zero-allocation-stringbuilder-for-net-core-and-unity-f3163c88c887
[^2]: ZString README, "Unity" section — minimum supported Unity 2021.3, UPM git URL install. https://github.com/Cysharp/ZString#unity
[^3]: ZString README, "License" — number-conversion methods under `ZString/Number` vendored from dotnet/runtime for .NET Standard 2.0 / Unity. https://github.com/Cysharp/ZString#license
[^4]: Cysharp/ZString repository metadata (created 2020-02-03). https://github.com/Cysharp/ZString

## Tags

csharp, dotnet, unity, string-builder, zero-allocation, performance, gc-optimization, arraypool, utf8, textmeshpro, formatting
