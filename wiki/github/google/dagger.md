# google/dagger

> Compile-time dependency injection for Java, Kotlin, and Android — no reflection, all wiring resolved and code-generated at build time.

[GitHub repo](https://github.com/google/dagger) ·
[Official website](https://dagger.dev) ·
[License: Apache-2.0](https://github.com/google/dagger/blob/master/LICENSE.txt)

## Overview

Dagger is a dependency injection framework that does all of its graph analysis at compile time and emits plain Java source code to do the wiring. It uses no runtime reflection and no runtime bytecode generation: an annotation processor reads your `@Inject`, `@Module`, and `@Component` declarations, validates the object graph, and generates the factory and component classes that construct your objects[^1]. If a dependency is missing or a cycle exists, the build fails with a compiler error rather than a runtime `ProvisionException`.

The project has a two-generation lineage that matters. Dagger 1 was written at Square (Jesse Wilson, Bob Lee) as a faster, more static alternative to Guice[^2]. Dagger 2, a ground-up rewrite led by Google (Gregory Kick and colleagues), moved *all* graph resolution to compile time and dropped the last runtime reflection that Dagger 1 retained[^3]. The `square/dagger` repository is Dagger 1 and is effectively frozen; `google/dagger` is Dagger 2 and is the only version under active development. Everything current — Hilt, Producers, KSP support — lives here.

The defining tradeoff is verbosity and build-time cost in exchange for correctness guarantees and zero runtime overhead. Dagger's generated code is what you would have hand-written, so provisioning is a plain method call with no reflective lookup. The price is a steep conceptual model (components, scopes, subcomponents, multibindings, qualifiers), error messages that can be dense, and annotation-processing time added to every build. This is why Hilt exists: an opinionated layer that hides most of the component wiring for Android apps.

## Getting Started

Gradle (Java or Kotlin with kapt/KSP):

```groovy
dependencies {
  implementation 'com.google.dagger:dagger:2.60.1'
  annotationProcessor 'com.google.dagger:dagger-compiler:2.60.1'
  // Kotlin: use kapt or, preferably, KSP:
  // ksp 'com.google.dagger:dagger-compiler:2.60.1'
}
```

A minimal graph:

```java
// A dependency with an @Inject constructor — Dagger knows how to build it.
class Engine {
  @Inject Engine() {}
}

class Car {
  private final Engine engine;
  @Inject Car(Engine engine) { this.engine = engine; }
}

// The component is the injector. Dagger generates DaggerCarComponent.
@Component
interface CarComponent {
  Car car();
}

public class Main {
  public static void main(String[] args) {
    CarComponent component = DaggerCarComponent.create();
    Car car = component.car();   // fully constructed graph, no reflection
  }
}
```

`@Module` + `@Provides` methods cover types you cannot annotate directly (interfaces, third-party classes, values needing custom construction).

## Architecture / How It Works

Dagger is an `javax.annotation.processing.Processor` (it also supports the `jakarta.inject` annotations and Kotlin's KSP). The pipeline runs entirely inside `javac`/`kapt`/KSP:

1. **Discovery** — collects `@Inject` constructors, `@Provides`/`@Binds` module methods, and `@Component`/`@Subcomponent` interfaces.
2. **Binding graph resolution** — builds a directed graph of every requested type to its binding. Missing bindings, dependency cycles (without `Provider`/`Lazy` to break them), and scope mismatches are reported as compile errors here.
3. **Code generation** — emits a `Dagger<ComponentName>` implementation plus `_Factory` classes. Each `@Inject` type gets a factory; the component composes them.

Core concepts:

- **Components / Subcomponents** — the object graph's roots. Subcomponents inherit their parent's bindings and add their own; this is how per-request or per-screen scopes nest inside an application scope.
- **Scopes** (`@Singleton`, `@Reusable`, custom) — a scoped binding is instantiated at most once per component instance that owns the scope. Dagger enforces that a scoped binding lives in a matching-scoped component.
- **`Provider<T>` and `Lazy<T>`** — wrap a binding to defer or repeat construction; also the sanctioned way to break dependency cycles.
- **Multibindings** (`@IntoSet`, `@IntoMap`) — collect contributions from many modules into a `Set`/`Map`, the standard plugin pattern.

Three layers ship on top of the core:

- **Hilt** (`dagger.hilt.*`) — an Android-specific opinionated layer that defines standard components (Application, Activity, Fragment, ViewModel) and generates the boilerplate for attaching them to the Android lifecycle. Most new Android apps use Hilt, not raw Dagger[^4].
- **Producers** (`dagger-producers`) — an asynchronous variant built on `ListenableFuture` for parallelizable, potentially I/O-bound graphs. Niche; used inside Google server code more than in apps.
- **`dagger.android`** — the older, pre-Hilt Android integration (`AndroidInjection`, `@ContributesAndroidInjector`). Still present but Hilt is the recommended path for new code.

## Production Notes

**Build-time cost is the real tax.** Annotation processing adds to every compile. On large multi-module Android builds this is significant; the standard mitigations are enabling Gradle's incremental annotation processing (Dagger's processor is isolating/incremental), splitting the graph across Gradle modules so a change doesn't reprocess everything, and — for Kotlin — migrating off `kapt` to **KSP**, which is markedly faster because it avoids kapt's Java-stub generation step. Dagger's KSP support is the recommended path for Kotlin projects going forward.

**Error messages have a learning curve.** A missing binding produces a wall of text listing the dependency chain. It is precise but intimidating; the key skill is reading from the bottom (the unsatisfied request) upward. `-Adagger.fullBindingGraphValidation=ERROR` and the SPI/`BindingGraph` plugin API let teams add custom validation, but the default output is what most people fight with.

**Kotlin friction.** `kapt` is in maintenance mode and slow; KSP is the answer but historically lagged kapt on some Dagger features, so verify feature parity for your version. `lateinit`/nullability, companion-object providers, and `@JvmStatic` on `@Provides` for performance are recurring Kotlin-specific gotchas.

**Scope and component leaks.** Holding a reference to an `@ActivityScoped` object past the Activity's life, or accidentally provisioning a scoped binding from the wrong component, leaks memory. Hilt's standardized components reduce this class of mistake but do not eliminate it.

**Upgrade path.** Dagger 1 (`square/dagger`) to Dagger 2 is a rewrite, not an upgrade — different annotations and no runtime graph. The `javax.inject` to `jakarta.inject` transition and Guice interop are handled but require attention on Jakarta EE stacks. Within Dagger 2 the API is very stable; version bumps are usually low-risk.

**No runtime configurability.** Because the graph is fixed at compile time, you cannot swap bindings via a config file at startup the way a reflection-based container allows. Test doubles are injected by replacing modules/components at compile time (Hilt provides `@TestInstallIn` and `@BindValue` for this).

## When to Use / When Not

**Use when:**
- You want dependency-injection errors caught at build time, not in production.
- You're building an Android app (Hilt is the de facto standard) or a Java service where startup speed and zero reflection matter.
- The object graph is large enough that manual wiring is error-prone but stable enough to be known at compile time.

**Avoid when:**
- You need runtime-dynamic graph reconfiguration (plugin systems that decide bindings from external config at startup) — a reflection container fits better.
- The project is small; a few constructors wired by hand cost less than Dagger's learning curve and build overhead.
- You want minimal ceremony in Kotlin and are willing to trade some maturity — `koin` or `kotlin-inject` are lighter.

## Alternatives

- google/guice — same lineage of ideas, but resolves the graph at runtime via reflection; more dynamic, slower startup, errors surface at runtime. Use when you need runtime-configurable bindings.
- InsertKoinIO/koin — Kotlin DSL service locator, no annotation processing, minimal build cost; use when you want simplicity and can accept runtime resolution.
- evant/kotlin-inject — compile-time DI designed for Kotlin/KSP with a lighter model; use for Kotlin-first projects wanting Dagger-style guarantees without Dagger's weight.
- spring-projects/spring-framework — full application container with DI plus much more; use when you're already in the Spring ecosystem.
- square/dagger — Dagger 1, the predecessor; do not use for new code, it is frozen.

## History

| Version | Date | Notes |
|---------|------|-------|
| Dagger 1 | 2012 | Square's original; partial runtime reflection[^2]. |
| 2.0 | 2015-04 | Google rewrite, fully compile-time, no reflection[^3]. |
| 2.x producers | 2015+ | `ListenableFuture`-based async graphs. |
| Hilt (alpha) | 2020-06 | Opinionated Android layer on top of Dagger[^4]. |
| KSP support | 2023 | Kotlin Symbol Processing backend as a kapt alternative. |
| 2.60.1 | 2026 | Current release line referenced by the README[^1]. |

## References

[^1]: Dagger README and documentation. https://dagger.dev and https://github.com/google/dagger
[^2]: Square's Dagger 1 (predecessor, now frozen). https://github.com/square/dagger
[^3]: Dagger 2 overview and design (no-reflection, compile-time graph). https://dagger.dev/dev-guide/
[^4]: Hilt — dependency injection for Android. https://dagger.dev/hilt/

## Tags

java, kotlin, android, dependency-injection, annotation-processor, compile-time, ksp, hilt, jvm, google
