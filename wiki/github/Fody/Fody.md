# Fody/Fody

> Build-time IL weaving for .NET — an MSBuild task plus a plugin model that rewrites your compiled assemblies with Mono.Cecil.

[GitHub repo](https://github.com/Fody/Fody) ·
[Documentation (Fody/Home)](https://github.com/Fody/Home) ·
[License: MIT](https://github.com/Fody/Fody/blob/master/License.txt)

## Overview

Fody is an extensible tool for manipulating the IL (Intermediate Language) of a .NET assembly as part of the build[^1]. After the C# (or F#/VB) compiler emits an assembly, Fody re-opens it with Mono.Cecil, hands it to a chain of add-ins called "weavers," and writes the modified assembly and PDB back to disk. The compiler never sees the transformation; the shipped DLL is different from what `csc` produced.

The value proposition is eliminating plumbing. Rewriting IL directly means learning both Mono.Cecil and the MSBuild/build-integration surface — a substantial amount of boilerplate before any actual transformation happens. Fody owns that plumbing (MSBuild task, add-in discovery, assembly load/save, PDB handling, symbol updating) so a weaver author only implements a `ModuleWeaver` class[^2]. The result is a large ecosystem of small, focused weavers: PropertyChanged.Fody (auto-implement `INotifyPropertyChanged`), Costura.Fody (embed dependencies into one assembly), NullGuard.Fody (null-argument checks), MethodTimer.Fody, Equals.Fody, ConfigureAwait.Fody, and dozens more.

The defining tension is that IL weaving is inherently invisible and post-compilation. The behavior of a woven assembly is not derivable from reading the source, which complicates debugging, code review, and reasoning about what actually ships. Since .NET 5, Roslyn source generators cover a growing share of the same use cases at the source level with first-party support — Fody's relevance now concentrates on transformations that *cannot* be expressed as additive source (rewriting existing method bodies, embedding assemblies, stripping/injecting IL).

## Getting Started

Fody itself does nothing; you install it alongside one or more weaver packages.

```bash
dotnet add package Fody
dotnet add package PropertyChanged.Fody
```

Fody auto-generates a `FodyWeavers.xml` at the project root on first build. It lists (and orders) active weavers:

```xml
<Weavers xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="FodyWeavers.xsd">
  <PropertyChanged/>
</Weavers>
```

With PropertyChanged.Fody active, this plain class raises change notifications automatically after weaving:

```csharp
using PropertyChanged;

[AddINotifyPropertyChangedInterface]
public class Person
{
    public string GivenName { get; set; }
    public string FamilyName { get; set; }
    public string FullName => $"{GivenName} {FamilyName}";  // notified when either name changes
}
```

The `Fody` package reference must set `PrivateAssets="all"` (the template does this) so it does not flow transitively to consumers.

## Architecture / How It Works

Fody is an MSBuild task that hooks in after the compile step. The pipeline:

1. **Add-in discovery** — Since Fody 3.0, weavers ship as NuGet packages named `*.Fody` that place a `<name>.Fody/<name>.dll` under their `weaver` folder[^3]. Fody resolves them from the restored package graph rather than an in-solution assembly copy.
2. **Configuration** — `FodyWeavers.xml` selects which discovered weavers run, in what order, with per-weaver element config. Order matters: weavers see the assembly in the state the previous weaver left it.
3. **Load** — The target assembly (and its PDB) is read into a Mono.Cecil `ModuleDefinition`[^4].
4. **Weave** — Each weaver's `ModuleWeaver.Execute()` mutates the `ModuleDefinition`: injecting types, rewriting method bodies, adding attributes, removing members.
5. **Write** — The modified module and updated symbols are written back over the compiler output. A marker type `ProcessedByFody` is injected so tools can detect a woven assembly.

Weaver authoring is the extensibility contract: implement `ModuleWeaver`, expose `Execute()`, declare which references to inject via `GetAssembliesForScanning()`. The core engine handles cross-cutting concerns — logging routed to MSBuild, symbol reading/writing, strong-name re-signing, and "in-solution weaving" for developing a weaver in the same solution that consumes it.

The coupling story: Fody sits between the compiler and the rest of the toolchain, and every downstream tool that inspects the assembly (debuggers, decompilers, analyzers, trimmers, other post-build steps) sees the woven output. That places Fody on the critical path of the build in a way source-level tools are not.

## Production Notes

**Tooling-upgrade fragility.** Fody rewrites assemblies via MSBuild integration and Mono.Cecil, both sensitive to SDK and MSBuild changes. Major .NET SDK bumps have historically required corresponding Fody/weaver updates; pinning the SDK (`global.json`) and upgrading Fody + weavers together is the safe path. A mismatched weaver against a new Cecil/SDK is a common breakage.

**Debugging woven code.** Because the shipped IL differs from source, stepping through woven methods can land on unexpected lines, and stack traces may reference injected members. PDB/sequence-point fidelity depends on the weaver behaving well; a buggy weaver produces confusing symbols. Review woven output with a decompiler (ILSpy/dotPeek) when behavior is surprising.

**Weaver ordering and interactions.** The order in `FodyWeavers.xml` is significant and not always obvious — e.g. a weaver that rewrites property setters must run before or after one that injects properties depending on intent. Combining many weavers increases the chance of one stepping on another's output.

**Build cost and determinism.** Weaving adds an assembly load/rewrite/save pass to every build of the project. It also interacts with incremental build and deterministic-build settings; caching layers that assume the compiler output is final can be fooled by post-compile rewriting.

**Licensing expectation.** The code is MIT, but the project explicitly states it "is expected that all developers using Fody become a Patron on OpenCollective"[^5]. This is a social/funding expectation layered on top of a permissive license, not an enforced legal term — worth surfacing to legal/compliance teams who scan for obligations, since it is unusual and easy to misread as a paid license.

**Source generators as the modern default.** For transformations that can be expressed as *added* code (partial classes, generated members), Roslyn source generators are first-party, debuggable at source level, and avoid post-compile rewriting. Reach for Fody when you must rewrite or remove existing IL, not merely add to it.

## When to Use / When Not

**Use when:**
- You need to rewrite existing method bodies or strip/inject IL that cannot be expressed as additive source.
- You want a battle-tested weaver that already exists (PropertyChanged, Costura, NullGuard) rather than writing codegen yourself.
- You need to embed dependencies into a single assembly (Costura.Fody) or apply cross-cutting behavior across many types.

**Avoid when:**
- The transformation is purely additive — a Roslyn source generator is better supported and debuggable.
- You need maximum build reproducibility and minimal moving parts in CI across SDK upgrades.
- Your team cannot own the debugging cost of code that differs from its source, or you ship to environments where post-compile assembly modification is disallowed.

## Alternatives

- dotnet/roslyn — source generators run inside the compiler and add source; use instead of Fody whenever the change is additive rather than a rewrite.
- jbevain/cecil — the Mono.Cecil IL library Fody wraps; use directly when you want full control over weaving without the MSBuild/plugin layer.
- postsharp/Metalama — Roslyn-based aspect framework (commercial tiers) with source-level debugging; use when you want a supported AOP product rather than assembling weavers.
- pardeike/Harmony — runtime method patching (monkey-patching) instead of build-time weaving; use for modding/instrumenting assemblies you do not build.
- MonoMod/MonoMod — runtime and build-time IL patching aimed at game modding; use when patching third-party assemblies you cannot recompile.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2012 | Fody created by Simon Cropp; MSBuild task + Mono.Cecil weaving[^1]. |
| 3.0 | 2018 | Weavers distributed as `*.Fody` NuGet packages; `FodyWeavers.xml` schema and NuGet-based add-in discovery[^3]. |
| 4.x–5.x | 2019–2020 | .NET Core/SDK build integration, ongoing Cecil and MSBuild task-host updates. |
| 6.x | 2021+ | Modernized MSBuild integration and tooling baseline; current major line. |

Release notes are tracked as GitHub Milestones rather than a single changelog file[^6]. Exact minor dates vary; the 3.0 add-in-packaging change is the migration most projects remember.

## References

[^1]: Fody README — "Extensible tool for weaving .net assemblies." https://github.com/Fody/Fody
[^2]: Fody addin development guide. https://github.com/Fody/Home/blob/master/pages/addin-development.md
[^3]: Fody addin discovery / packaging. https://github.com/Fody/Home/blob/master/pages/addin-packaging.md
[^4]: Mono.Cecil — the IL manipulation library Fody builds on. https://github.com/jbevain/cecil
[^5]: Fody licensing and patron FAQ. https://github.com/Fody/Home/blob/master/pages/licensing-patron-faq.md
[^6]: Fody release milestones. https://github.com/Fody/Fody/milestones?state=closed

## Tags

dotnet, csharp, il-weaving, msbuild, mono-cecil, aop, build-tools, code-generation, assembly-manipulation, nuget
