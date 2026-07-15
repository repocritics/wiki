# rrousselGit/freezed

> Dart code generator for immutable data classes, tagged unions, `copyWith`, and JSON — trades build-step overhead for eliminated boilerplate.

[GitHub repo](https://github.com/rrousselGit/freezed) ·
[pub.dev package](https://pub.dev/packages/freezed) ·
[License: MIT](https://github.com/rrousselGit/freezed/blob/master/packages/freezed/LICENSE)

## Overview

Freezed is a Dart build-time code generator that writes the mechanical parts of a
"model" class for you: the constructor-to-field mapping, `==`/`hashCode`,
`toString`, `copyWith`, and — via `json_serializable` — `fromJson`/`toJson`.
It also generates tagged unions (sealed class hierarchies) from multiple factory
constructors. It is authored by Rémi Rousselet, who also maintains Riverpod and
provider, and is one of the most widely used packages in the Flutter ecosystem,
carrying the Flutter Favorite designation[^1].

The defining tension is codegen versus language features. Freezed exists because
Dart lacks primary constructors, data classes, and (historically) sum types, so a
hand-written immutable model runs to dozens of error-prone lines. Freezed removes
that, but at the cost of a `build_runner` step, `part` files, and generated
`.freezed.dart` / `.g.dart` artifacts that must stay in sync with your source. As
Dart itself gained sealed classes, pattern matching, and switch expressions in
Dart 3, part of Freezed's original value (its own `when`/`map` matchers) was
superseded by the language — which is precisely what the 3.0.0 release leaned
into[^2].

## Getting Started

```console
# Flutter project
flutter pub add freezed_annotation dev:build_runner dev:freezed
# add these too if you want fromJson/toJson:
flutter pub add json_annotation dev:json_serializable
```

```dart
// person.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'person.freezed.dart';
part 'person.g.dart'; // only needed for JSON

@freezed
abstract class Person with _$Person {
  const factory Person({
    required String firstName,
    required String lastName,
    required int age,
  }) = _Person;

  factory Person.fromJson(Map<String, Object?> json) => _$PersonFromJson(json);
}
```

```console
dart run build_runner watch -d   # regenerates on save; use `build` for one-shot
```

## Architecture / How It Works

Freezed is a `source_gen` `Builder` driven by `build_runner`. It reads the
analyzer's resolved AST for any library that declares a matching `part`
directive, finds classes annotated `@freezed` / `@unfreezed` / `@Freezed`, and
emits an augmenting mixin (`_$Person`) plus the concrete implementation classes
(`_Person`) into a sibling `.freezed.dart` file. The `part` mechanism is load
bearing: the generated code shares the library's private scope, which is how the
`_$`-prefixed mixin can implement your public class. JSON is delegated — Freezed
does not serialize anything itself; it wires up calls to code produced by the
separate `json_serializable` generator, which is why serializable models need
both a `.freezed.dart` and a `.g.dart` part.

Two class shapes exist. **Primary constructors** (a `factory` redirecting to a
generated `_Foo`) are the common path and produce fully immutable classes.
**Classic classes** (a normal constructor with `final` fields) let Freezed add
only `copyWith`/`==`/`toString` while you keep control of inheritance and
non-constant defaults. Mutable models use `@unfreezed`, which drops the
value-equality override — a deliberate consequence, since identity equality is
the only coherent choice for a mutable object.

Unions are just a class with more than one factory constructor. Before Dart 3,
Freezed generated `when`/`map`/`maybeWhen` helpers to branch over the variants.
Freezed 3.0.0 (2025-02-25) rebased unions onto native Dart 3 `sealed`/`abstract`
classes so you branch with the language's own exhaustive `switch` expressions and
pattern matching; the old generated matchers are retained as a legacy path but are
no longer the recommended API[^2]. `copyWith` supports deep-copy chaining
(`company.copyWith.director.assistant(name: ...)`) when nested fields are
themselves Freezed models.

## Production Notes

- **The build step is the cost.** On large models or large projects,
  `build_runner` codegen is slow and CPU-heavy, and it must re-run on every model
  change. Teams keep `build_runner watch` running during development; CI needs an
  explicit `dart run build_runner build --delete-conflicting-outputs` before
  analyze/test. Forgetting to regenerate is the single most common source of
  confusing "member not found" errors.
- **Generated files are real files.** `.freezed.dart`/`.g.dart` inflate the repo
  (commit them or regenerate in CI — pick one and be consistent) and add to
  analyzer load, which can slow IDE responsiveness in model-heavy codebases.
- **The 2.x → 3.0 migration is not mechanical.** 3.0 requires marking union/model
  classes `sealed`/`abstract`, and moves you off generated `when`/`map` toward
  native pattern matching. Exhaustive-switch call sites and mixed old/new syntax
  are the usual breakage; the migration guide is required reading before
  upgrading[^3].
- **`json_serializable` coupling leaks.** Recent `json_serializable`/`meta`
  versions may force you to silence `invalid_annotation_target` in
  `analysis_options.yaml`. Version-skew between `freezed`, `freezed_annotation`,
  `json_serializable`, and `build_runner` is a recurring pubspec headache.
- **Collections are made unmodifiable by default** under `@freezed`, so mutating a
  `List`/`Map`/`Set` field throws at runtime; opt out with
  `@Freezed(makeCollectionsUnmodifiable: false)` if you need it.
- **`factory` constraints** mean asserts, non-const defaults, and `super()` calls
  require the `@Assert`/`@Default` annotations or a private `MyClass._()`
  constructor — an inconvenience that surfaces on any non-trivial model.

## When to Use / When Not

**Use when:**
- You have many models needing value equality, `copyWith`, and JSON, and want the
  boilerplate gone.
- You model domain state as sum types (loading/data/error) and want exhaustive
  switch coverage checked by the compiler.
- You already run `build_runner` for other generators, so the build step is not a
  new tax.

**Avoid when:**
- Your models are few and simple — Dart 3 sealed classes, records, and a
  hand-written `copyWith` may be cheaper than adopting a generator.
- You cannot tolerate a codegen step in your build/CI, or want zero generated
  artifacts.
- You need serialization features beyond what `json_serializable` covers, or want
  a single package that does models and JSON without part files.

## Alternatives

- schultek/dart_mappable — use instead when you want unions, polymorphism, and JSON in one generator with no `part` files.
- google/built_value — use instead when you want an established immutable-value library and accept builder-based boilerplate.
- dart-lang/json_serializable — use instead when you only need JSON (de)serialization, not `copyWith` or unions (Freezed itself depends on it).
- Native Dart 3 `sealed` classes + records — use instead when models are simple enough that codegen overhead outweighs the boilerplate saved.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2020-01 | Initial release; `build_runner` codegen for data classes and unions[^1]. |
| 2.0.0 | 2022-05 | Major API revision; `@Default`/`@Assert`, deep-copy `copyWith`. |
| 3.0.0 | 2025-02-25 | Rebased unions on Dart 3 `sealed`/`abstract` classes and native pattern matching; legacy `when`/`map` retained but de-emphasized[^2]. |

## References

[^1]: freezed on pub.dev — package page, Flutter Favorite badge, install and usage docs. https://pub.dev/packages/freezed
[^2]: freezed CHANGELOG — 3.0.0 (2025-02-25). https://github.com/rrousselGit/freezed/blob/master/packages/freezed/CHANGELOG.md#300---2025-02-25
[^3]: freezed 2.x → 3.0 migration guide. https://github.com/rrousselGit/freezed/blob/master/packages/freezed/migration_guide.md

## Tags

dart, flutter, code-generation, build-runner, immutability, data-classes, union-types, sealed-classes, json-serialization, copywith
