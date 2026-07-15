# diffplug/spotless

> A build-tool plugin that aggregates existing code formatters behind one `check`/`apply` interface for Gradle, Maven, and sbt.

[GitHub repo](https://github.com/diffplug/spotless) ·
[License: Apache-2.0](https://github.com/diffplug/spotless/blob/main/LICENSE)

## Overview

Spotless is not itself a code formatter. It is an orchestration layer that wraps
formatters other people wrote — google-java-format, Eclipse JDT, ktlint, ktfmt,
scalafmt, Prettier, ESLint, Black, clang-format, gofmt, shfmt, buf, Biome, and
several dozen more — and exposes them through two verbs: check (fail the build on
violations) and apply (rewrite the files in place). It is maintained by DiffPlug
and has been in development since 2015[^1]. The value proposition is integration
glue, not formatting logic: a formatter is conceptually just a
`Function<String, String>`, but wiring one into a build so it handles line
endings, encodings, idempotency, incremental up-to-date checks, and per-language
targeting correctly is the tedious part Spotless owns[^2].

The defining tension is that Spotless inherits every quirk of every tool it
wraps. It gives you a uniform build interface, but the underlying formatter still
has its own version, its own bugs, and its own runtime requirements (a JVM
classpath for Java formatters, a Node install for the Prettier/ESLint steps). A
single `spotlessApply` can shell out to npm, spin up isolated classloaders, and
run native binaries — so "one plugin" hides a fairly heterogeneous set of moving
parts. Its most-used surface by far is the Java + Kotlin + Gradle path; the long
tail of supported languages varies in maturity and in which build plugin
(Gradle vs. Maven) actually exposes a given step.

## Getting Started

Gradle (`build.gradle`):

```groovy
plugins {
  id 'com.diffplug.spotless' version '7.0.2'
}

spotless {
  java {
    googleJavaFormat('1.25.2')   // pin the formatter version explicitly
    removeUnusedImports()
    importOrder()
  }
}
```

```console
$ ./gradlew spotlessCheck   // fails the build on any violation
$ ./gradlew spotlessApply   // rewrites files to conform
```

Maven (`pom.xml`), invoked as `mvn spotless:check` / `mvn spotless:apply`:

```xml
<plugin>
  <groupId>com.diffplug.spotless</groupId>
  <artifactId>spotless-maven-plugin</artifactId>
  <version>2.44.0</version>
  <configuration>
    <java><googleJavaFormat><version>1.25.2</version></googleJavaFormat></java>
  </configuration>
</plugin>
```

## Architecture / How It Works

The core abstraction is `FormatterStep`: a named, serializable transform from
unformatted string to formatted string. A `Formatter` is an ordered list of steps
applied to a file; the build plugins (`plugin-gradle`, `plugin-maven`, plus an
external sbt plugin) are thin adapters that discover target files, feed them
through the step list, and either diff (check) or overwrite (apply)[^2].

Steps that need a third-party library do not put it on your build classpath.
Spotless resolves each formatter's dependencies through a "provisioner" (Gradle's
or Maven's dependency resolution) and loads them into an **isolated classloader**.
This is what lets Spotless run, say, google-java-format 1.25 without colliding
with whatever Guava your build already uses, and lets two projects pin different
formatter versions. The npm-backed steps (Prettier, ESLint, tsfmt) are different:
they require a Node.js/npm install on the machine, materialize a `node_modules`,
and communicate with a spawned Node process. Native steps (clang-format, gofmt,
shfmt, buf) shell out to an installed binary.

Two safeguards are baked in. **PADDEDCELL** is an idempotency check: some
formatters are not idempotent (applying twice yields a different result), and
Spotless detects the cycle and reports a diagnostic rather than silently
oscillating[^3]. The **encoding safeguard** surfaces a readable error when a file
is misdecoded rather than corrupting it. Line endings are resolved via git
attributes by default, so cross-platform CRLF/LF differences don't produce
phantom violations.

Two performance features matter in practice. **Incremental / up-to-date
checking** lets Gradle skip files that haven't changed, and on Gradle the check
result participates in the build cache. **Ratchet** (`ratchetFrom 'origin/main'`)
restricts formatting to files that differ from a git ref, which is how large
legacy codebases adopt Spotless without a single reformat-the-world commit — you
only enforce formatting on lines you touch.

## Production Notes

- **Formatter versions drift with Spotless upgrades.** If you don't pin a
  version, Spotless uses its own default, and bumping the Spotless plugin can
  silently change the default formatter version — producing a large "reformat
  everything" diff that has nothing to do with your code. Always pin the inner
  formatter version explicitly (as in the examples above) and upgrade it
  deliberately.
- **google-java-format vs. the JDK module system.** google-java-format reaches
  into internal `com.sun.tools.javac` packages. On JDK 16+ this requires
  `--add-exports`/`--add-opens` JVM args for the process running Spotless, or the
  step fails with an `IllegalAccessError`. This is the single most common Java
  support issue; palantir-java-format is often chosen partly to avoid it.
- **npm steps are the slow, fragile path.** Prettier/ESLint/tsfmt need Node on
  PATH, do a network install on first run, and add real latency. In CI, cache
  `node_modules` and the npm download, or expect multi-minute cold starts. These
  steps are also the most likely to break on hermetic/offline builders.
- **Ratchet needs real git history.** `ratchetFrom` uses JGit; shallow clones
  (`--depth`) or CI checkouts that don't fetch the base ref will fail or
  mis-detect changed files. Ensure the comparison ref is present locally.
- **Gradle configuration cache.** Compatibility has improved over time but has
  historically been incomplete for some steps; if you enable the configuration
  cache and hit failures, suspect a specific formatter step before blaming your
  own build.
- **check is not a linter report.** `spotlessCheck` fails fast and prints a diff;
  it does not annotate PRs or produce a rules report. Pair it with a pre-push or
  CI gate, and expect contributors to run `spotlessApply` locally.

## When to Use / When Not

**Use when:**
- You have a polyglot JVM codebase (Java + Kotlin + Groovy + config files) and
  want one `apply` command instead of a formatter plugin per language.
- You want to adopt formatting on a large legacy repo incrementally via ratchet.
- You want formatting enforced by the build/CI, version-pinned and reproducible.

**Avoid when:**
- Your project is pure JavaScript/TypeScript/CSS — run prettier/eslint directly;
  Spotless's npm bridge only adds a layer of indirection and latency.
- You only need one formatter for one language (e.g. Java-only) and don't want an
  aggregation framework — call that formatter's own plugin.
- You want lint *reporting* and rule enforcement (unused-variable, complexity)
  rather than auto-formatting — that is Checkstyle/PMD/ESLint-as-linter territory.

## Alternatives

- prettier/prettier — if your codebase is JS/TS/CSS/Markdown only, use Prettier
  directly; Spotless's Node bridge exists for mixed repos, not JS-first ones.
- google/google-java-format — when you only need Java formatting and prefer the
  formatter's own Gradle/Maven integration without the aggregation layer.
- pinterest/ktlint — standalone Kotlin lint-and-format; Spotless wraps it, but a
  Kotlin-only project can run it directly.
- pre-commit/pre-commit — language-agnostic git-hook orchestration if you prefer
  hooks over build-task gating and don't want a JVM build plugin.
- checkstyle/checkstyle — use when you need rule enforcement and reporting rather
  than automatic rewriting; complementary to, not a replacement for, Spotless.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-04 | Repository created; project begins as a Gradle-first formatter plugin[^1]. |
| Maven plugin | ~2019 | `spotless-maven-plugin` adds a second build-system target. |
| 6.0 | 2021 | Major release; raised minimum JRE/Gradle baselines[^4]. |
| 7.0 | 2025 | Major release; baseline and formatter-default updates[^4]. |

## References

[^1]: diffplug/spotless repository metadata (created 2015-04-27; Apache-2.0;
~5.5k stars as of 2026-07). https://github.com/diffplug/spotless
[^2]: Spotless README, "How it works (for potential contributors)" — describes
the `Function<String, String>` model and the integration concerns Spotless owns.
https://github.com/diffplug/spotless#how-it-works-for-potential-contributors
[^3]: PADDEDCELL idempotency documentation.
https://github.com/diffplug/spotless/blob/main/PADDEDCELL.md
[^4]: Spotless changelog (per-plugin CHANGES files); consult for exact version
dates and baseline changes. https://github.com/diffplug/spotless/blob/main/CHANGES.md

## Tags

java, gradle, maven, code-formatter, linter, build-plugin, jvm, kotlin, prettier, formatting, developer-tooling
