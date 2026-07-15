# square/kotlinpoet

> A builder-based Kotlin/Java API for generating `.kt` source files, with automatic import management.

[GitHub repo](https://github.com/square/kotlinpoet) ·
[Official website](https://square.github.io/kotlinpoet/) ·
[License: Apache-2.0](https://github.com/square/kotlinpoet/blob/main/LICENSE.txt)

## Overview

KotlinPoet is a library for emitting Kotlin source code programmatically. You describe the output as a tree of immutable spec objects — a `FileSpec` containing `TypeSpec`s, `FunSpec`s, `PropertySpec`s — and call `writeTo` to render formatted `.kt` text. It is the Kotlin sibling of JavaPoet[^1], sharing the same builder model, and is maintained by Square (Block). Created in 2017, it reached 1.0 in 2018 and 2.0 in 2024; it remains actively maintained, with releases every few months and a push date in July 2026.

Its primary consumers are code generators, not application developers: KSP symbol processors and annotation-processing tools that turn schemas, annotations, or IDL into Kotlin. Moshi's codegen, Wire, Room's Kotlin output, and many KSP-based libraries build on it. If you have ever seen a generated `_YourClassJsonAdapter.kt`, KotlinPoet likely wrote it.

The defining tension is that KotlinPoet models Kotlin's *structure*, not its *semantics*. It will happily emit code that does not compile — undeclared references, contradictory modifiers, type mismatches. It guarantees well-formatted, correctly-imported text; it does not guarantee valid programs. That trade keeps the API small and predictable, but pushes correctness onto the caller and the downstream Kotlin compiler.

## Getting Started

```kotlin
// build.gradle.kts
dependencies {
  implementation("com.squareup:kotlinpoet:2.3.0")
}
```

```kotlin
import com.squareup.kotlinpoet.*

val greeter = TypeSpec.classBuilder("Greeter")
  .primaryConstructor(
    FunSpec.constructorBuilder()
      .addParameter("name", String::class)
      .build()
  )
  .addProperty(
    PropertySpec.builder("name", String::class)
      .initializer("name")
      .build()
  )
  .addFunction(
    FunSpec.builder("greet")
      .addStatement("println(%P)", "Hello, \$name")
      .build()
  )
  .build()

FileSpec.builder("com.example", "Greeter")
  .addType(greeter)
  .build()
  .writeTo(System.out)
```

## Architecture / How It Works

The API is a set of immutable spec types, each with a nested `Builder`. Modify an existing spec by calling `toBuilder()`, mutating, and rebuilding — you never edit in place.

- **`FileSpec`** — a single `.kt` file. Owns the package declaration, top-level members, and the import table.
- **`TypeSpec`** — a class, object, interface, enum, annotation, or `companion object`. Holds properties, functions, nested types, supertypes, and modifiers.
- **`FunSpec` / `PropertySpec` / `ParameterSpec` / `TypeAliasSpec` / `AnnotationSpec`** — the member-level specs.
- **`CodeBlock`** — the leaf. Function bodies, initializers, and default values are `CodeBlock`s assembled from format strings with `addStatement`, `beginControlFlow`/`endControlFlow`, and `add`.

The **`TypeName`** hierarchy is the type model: `ClassName` (a fully-qualified type), `ParameterizedTypeName` (`List<String>`), `TypeVariableName` (generics), `WildcardTypeName`, `LambdaTypeName`, and `MemberName` (a top-level function or property). These carry nullability and annotations.

Rendering is driven by **format placeholders** inside `CodeBlock` and spec builders[^2]:

- `%S` — a string, quoted and escaped.
- `%L` — a literal, emitted verbatim (numbers, other specs, raw text).
- `%T` — a `TypeName`; **this is what makes imports automatic**. KotlinPoet collects every `%T`, adds imports to the `FileSpec`, and renders the short name; on collision it either aliases or fully-qualifies.
- `%N` — a named reference to another spec, keeping names consistent.
- `%M` — a `MemberName`; imports and references a top-level function/property.
- `%P` — a string template (interpolation preserved); `%%` — a literal percent.

The key architectural consequence: import management only works for types routed through `%T`/`%M`. Type references hand-built as `%L` strings bypass it and will not be imported — a frequent cause of "generated code references a type with no import."

Two interop modules extend the core. **`kotlinpoet-metadata`** reads Kotlin's `@Metadata` annotation (via `kotlin-metadata-jvm`) to reconstruct declarations from compiled class files. **`kotlinpoet-ksp`** bridges KSP: it converts `KSType`/`KSClassDeclaration` into KotlinPoet `TypeName`s and threads originating files for incremental compilation.

## Production Notes

**No semantic validation.** KotlinPoet is a structured text builder. Wrong modifiers, missing supertype constructors, references to names you never declared — all emit without complaint and fail later at `kotlinc`. Test generated code by actually compiling it (e.g. with `kotlin-compile-testing`), not by asserting on strings alone.

**`%S` vs `%L` is the top recurring bug.** `%S` wraps and escapes; `%L` is raw. Passing a value string to `%L` (unquoted, unescaped) or an identifier to `%S` (quoted when it shouldn't be) produces broken output that looks plausible. Reach for `%T`/`%N`/`%M` before falling back to raw `%L`.

**KSP incremental correctness.** In a symbol processor you must attach originating files — `addOriginatingKSFile(...)` on the spec, or `writeTo(codeGenerator, aggregating = ...)` — so KSP can invalidate generated files when inputs change[^3]. Omit it and you get stale generated code on incremental builds that only reappears after a clean.

**`kotlinpoet-metadata` is compiler-version-coupled.** It parses `@Metadata`, which only exists on Kotlin-compiled classes (never plain Java) and whose binary format is versioned. Reading metadata written by a much newer Kotlin compiler than your `kotlin-metadata-jvm` can fail or under-report. Pin versions deliberately.

**Language-feature lag.** New Kotlin syntax (context receivers/parameters, evolving `value class` rules) needs explicit modeling in KotlinPoet. Until a release adds it, you emit that syntax as raw `CodeBlock` strings, forfeiting import management and type safety for those fragments.

**2.0 was a breaking cleanup.** The 2.x line (2024) removed long-deprecated APIs and migrated metadata handling onto `kotlin-metadata-jvm`. Upgrading from 1.x is mostly mechanical but is not a drop-in bump; audit removed symbols. Within a major line the spec API is stable.

**Performance is rarely the issue.** Generation is build-time and fast for typical outputs. Very large single files cost some formatting time, but codegen wall-clock is usually dominated by the surrounding annotation-processing/KSP round, not KotlinPoet.

## When to Use / When Not

**Use when:**
- You are writing a KSP processor or annotation-processing tool that emits Kotlin.
- You generate Kotlin from a schema, IDL, or API description at build time.
- You want correct formatting and automatic, collision-safe imports without string-wrangling.

**Avoid when:**
- You need Java output — use JavaPoet instead.
- You need to *parse or transform* existing source — KotlinPoet only writes new files; it has no reader/AST-rewrite side. Use compiler plugins, KSP, or PSI.
- You need runtime/bytecode generation — use ASM or ByteBuddy.
- Your output is trivial and import-free — a plain string template is simpler.

## Alternatives

- square/javapoet — the Java-source counterpart with the same builder model; use it when the target is `.java`, not `.kt`.
- google/ksp — the symbol-processing front-end you usually pair *with* KotlinPoet; its own `CodeGenerator` suffices for simple text output where you don't need type modeling.
- JetBrains/kotlin — compiler plugins / FIR and `kotlin-metadata-jvm`; use when you must transform or read existing declarations rather than emit new files.
- pinterest/ktlint — not a generator but the formatter/linter commonly run over generated output when house style must match hand-written code.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2017 | First public release, forked from JavaPoet's model[^1]. |
| 1.0.0 | 2018 | Stable API. |
| 1.9.0 | 2021-06 | Metadata/interop improvements. |
| 1.12.0 | 2022-06 | Ongoing spec + KSP interop work. |
| 1.14.0 | 2023-05 | KSP interop refinements. |
| 1.16.0 | 2024-01 | Late 1.x maintenance line. |
| 1.18.1 | 2024-07 | Final 1.x release. |
| 2.0.0 | 2024-10 | Major: deprecated-API removal, `kotlin-metadata-jvm` migration[^4]. |
| 2.1.0 | 2025-02 | Post-2.0 feature release. |
| 2.2.0 | 2025-05 | Feature release. |
| 2.3.0 | 2026-03 | Current release line. |

## References

[^1]: KotlinPoet README — "a Kotlin and Java API for generating `.kt` source files." https://github.com/square/kotlinpoet
[^2]: KotlinPoet documentation, "Code & Control Flow" / placeholders (`%S`, `%L`, `%T`, `%N`, `%M`, `%P`). https://square.github.io/kotlinpoet/
[^3]: KotlinPoet KSP interop docs — originating files and incremental processing. https://square.github.io/kotlinpoet/interop-ksp/
[^4]: KotlinPoet releases (tags 2.0.0–2.3.0). https://github.com/square/kotlinpoet/releases

## Tags

kotlin, code-generation, source-generation, ksp, annotation-processing, jvm, metaprogramming, builder-pattern, square, developer-tools
