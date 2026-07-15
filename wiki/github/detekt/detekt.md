# detekt/detekt

> Static code analysis for Kotlin — a configurable rule engine over the Kotlin compiler's AST, with baselines and SARIF/HTML/XML reporting.

[GitHub repo](https://github.com/detekt/detekt) ·
[Official website](https://detekt.dev) ·
[License: Apache-2.0](https://github.com/detekt/detekt/blob/main/LICENSE)

## Overview

detekt is a static analyzer for Kotlin: it parses source into the Kotlin compiler's AST and runs a set of rules (visitors) that flag code smells, complexity, potential bugs, naming issues, and style violations. It began as a personal project by Artur Bosch — the still-current Maven group id `io.gitlab.arturbosch.detekt` records its GitLab origin — before moving to the `detekt` GitHub organization and gaining a team of maintainers. At roughly 7,000 stars and 847 forks it is the de facto community linter for Kotlin outside of formatting-only tools[^1].

The defining tension is **AST analysis vs. type resolution**. Most rules operate purely on the syntax tree, which is fast and needs no compiled classpath. A meaningful subset — anything that must know an expression's actual type (e.g. detecting a redundant `?.` on a non-null type) — requires *type resolution*, which means detekt has to run the Kotlin compiler's frontend against your full classpath. Type-resolution runs are slower, order-of-magnitude heavier on memory, and only work when the module actually compiles. Teams routinely run detekt in the cheap AST-only mode and never enable the rules that would catch the more interesting problems.

The second structural fact is **coupling to the Kotlin compiler**. detekt depends on `kotlin-compiler-embeddable`, so a given detekt release is pinned to a Kotlin frontend version. This is why the in-progress 2.0 line (alpha as of 2026) is a large rewrite onto the K2 compiler and its Analysis API, and why it also changes coordinates to the `dev.detekt` group — a migration users should plan for, not stumble into[^2].

## Getting Started

Gradle plugin (the first-party, most common path):

```kotlin
// build.gradle.kts
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.8"
}

detekt {
    buildUponDefaultConfig = true
    config.setFrom("$projectDir/config/detekt.yml")
    baseline = file("$projectDir/config/baseline.xml")
}
```

```sh
./gradlew detekt                    # run analysis
./gradlew detektBaseline            # snapshot existing issues to baseline.xml
./gradlew detektGenerateConfig      # write a default config/detekt.yml
```

Standalone CLI (no build tool):

```sh
curl -sSLO https://github.com/detekt/detekt/releases/download/v1.23.8/detekt-cli-1.23.8-all.jar
java -jar detekt-cli-1.23.8-all.jar --input src/ --report html:report.html
```

## Architecture / How It Works

detekt reuses the Kotlin compiler's front end. Source files are parsed into **PSI** (Program Structure Interface) trees — the same AST IntelliJ and the compiler use. A rule is a class extending `Rule` that implements `visit*` callbacks over PSI nodes; when it finds a violation it reports a `Finding` with a source location.

- **Rule sets.** Rules ship grouped into built-in sets: `comments`, `complexity`, `coroutines`, `empty-blocks`, `exceptions`, `naming`, `performance`, `potential-bugs`, and `style`. Each set and each rule is toggled and tuned in `detekt.yml`. A separate `formatting` rule set wraps pinterest/ktlint so detekt can also enforce (and auto-correct) formatting.
- **Type resolution.** Rules that need types are marked as requiring full analysis; detekt only runs them when given a `classpath` and JDK/Kotlin `jvmTarget`, having compiled a `BindingContext`. Without that context those rules are silently skipped — a frequent source of "why didn't detekt catch this" confusion.
- **Baselines.** `baseline.xml` records every finding present at a point in time as a suppression list, so a legacy codebase can adopt detekt with a clean gate while still failing the build on *new* smells. Findings are matched by an ID string (rule + signature), which can drift when code moves.
- **Suppression.** `@Suppress("RuleName")` in source is honored, matching the Kotlin/IntelliJ suppression mechanism.
- **Reports.** HTML (human), Markdown, XML in Checkstyle format (Jenkins/CI), and SARIF (GitHub Code Scanning). Reports are pluggable, as are custom rule sets loaded via the `detektPlugins` configuration.

Extensibility is a first-class design goal: third-party rule authors publish plugins (the `detekt-plugin` GitHub topic, indexed on the "Detekt Marketplace"), and the same extension points power the ktlint wrapper and the community rule packs.

## Production Notes

- **Type resolution cost is real.** Enabling `classpath`-backed rules can multiply runtime and memory; on large multi-module Android builds it is common to see the detekt task dominate CI time or OOM. Tune JVM heap for the Gradle worker and consider running full-analysis rules on a schedule rather than every PR.
- **Kotlin/detekt version lockstep.** Because detekt embeds a Kotlin compiler, upgrading Kotlin often forces a detekt upgrade (and vice versa). The README's compatibility table is the source of truth; mismatches produce obscure `NoSuchMethodError`/PSI failures, not clean errors.
- **AGP and Android.** On Android projects the `detekt` task does not automatically see variant source sets or generated code; wiring `detektMain`/per-variant tasks and pointing at the right `classpath` is manual. The Gradle plugin's defaults analyze `src/main/kotlin`-style layouts, not everything the Android plugin knows about.
- **Baseline rot.** Baselines suppress by finding ID; large refactors invalidate IDs and can either resurface old findings or hide new ones. Regenerate baselines deliberately, and review the diff — an accidental `detektBaseline` can silently whitelist real regressions.
- **`jvmTarget` must be set explicitly** on `Detekt` and `DetektCreateBaselineTask` tasks for type-resolution correctness; the README calls this out specifically.
- **2.0 is not drop-in.** The alpha changes the Maven group to `dev.detekt`, moves onto K2, and revises rule/API surfaces. Custom rules written against 1.x will need porting. Do not pin production gates to alpha builds.

## When to Use / When Not

**Use when:**
- You want configurable *code-smell* and complexity analysis for Kotlin, beyond formatting.
- You need a CI gate with baselines so a legacy codebase can adopt linting incrementally.
- You want SARIF output for GitHub Code Scanning, or Checkstyle XML for Jenkins.
- You want to write project-specific rules as first-class plugins.

**Avoid when:**
- You only want auto-formatting and canonical style — pinterest/ktlint or spotless is lighter and less to configure.
- Your codebase is not primarily Kotlin — detekt is Kotlin-only by design.
- You need the deepest type-aware checks but cannot afford full-classpath analysis time in CI.
- You require a stable plugin API today and cannot absorb the 1.x → 2.0 migration on the horizon.

## Alternatives

- pinterest/ktlint — opinionated formatter/linter with near-zero configuration; use when you want canonical style and auto-format, not smell detection (detekt wraps it as a rule set).
- diffplug/spotless — multi-language formatting orchestrator (can drive ktlint); use for build-time format enforcement rather than analysis.
- JetBrains/qodana — commercial IntelliJ-inspection-based static analysis platform; use when you want IDE-grade inspections and a hosted dashboard.
- SonarSource sonar-kotlin — SonarQube's Kotlin analyzer; use when analysis must live inside an existing Sonar quality-gate ecosystem.
- Android lint (in AGP) — use for Android-framework-specific correctness/API checks that detekt does not model.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2019-08 | First stable release after a long RC series; moved from GitLab to the detekt org[^1]. |
| 1.9–1.10 | 2020 | Type resolution support matured; SARIF reporting added. |
| 1.16 | 2021 | Config validation and rule-set improvements. |
| 1.23.0 | 2023 | Latest stable minor line; ongoing 1.23.x patch releases. |
| 1.23.8 | 2025 | Current stable, Kotlin 2.0.x compatible per compatibility table[^3]. |
| 2.0.0-alpha | 2025–2026 | K2 / Analysis API rewrite; group id → `dev.detekt` (in progress)[^2]. |

## References

[^1]: detekt repository and README, detekt/detekt. https://github.com/detekt/detekt
[^2]: detekt 2.0 alpha release notes and migration guidance, detekt.dev changelog. https://detekt.dev/changelog.html
[^3]: detekt compatibility / recommended-versions table. https://detekt.dev/docs/gettingstarted/gradle

## Tags

kotlin, static-analysis, linter, code-quality, gradle-plugin, jvm, code-smells, sarif, ci, developer-tools
