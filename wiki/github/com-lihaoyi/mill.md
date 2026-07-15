# com-lihaoyi/mill

> A JVM build tool for Java, Scala, and Kotlin whose build files are ordinary programs and whose task graph is content-addressed and cached.

[GitHub repo](https://github.com/com-lihaoyi/mill) ·
[Official website](https://mill-build.org/) ·
[License: MIT](https://github.com/com-lihaoyi/mill/blob/main/LICENSE)

## Overview

Mill is a build tool for the JVM ecosystem created by Li Haoyi, author of the
`com.lihaoyi` library suite (os-lib, uPickle, Ammonite, Requests-Scala)[^1]. It
positions itself against the two incumbents it is usually measured against:
simpler to configure than Maven, and easier to reason about than Gradle, while
claiming 3-7x faster dev-loop times through aggressive caching and default
parallelism[^2]. Its origins are in the Scala world, but the project has
deliberately broadened to treat Java and Kotlin as first-class targets.

The defining idea is that a build is a **typed object graph**, not a config
file. Modules are objects that extend traits (`JavaModule`, `ScalaModule`,
`KotlinModule`), and every build step is a node in a directed acyclic graph of
tasks. Each task's output is cached on disk keyed by its inputs, so a rebuild
only re-executes the tasks whose inputs changed — Bazel-style incrementality
applied to a much smaller, JVM-focused, single-machine tool.

The historical tension is that programmable builds cost you a learning curve: a
`build.mill` is Scala source, which means editing it well wants IDE support (via
BSP) and passing familiarity with the language even for a pure-Java project. Mill
1.0 partially addresses this with a declarative `build.mill.yaml` form for simple
cases, so teams can start without Scala and drop down to code when needed[^3].

## Getting Started

```bash
brew install mill          # macOS; or download the ./mill bootstrap launcher
mill init                  # import an existing Maven/Gradle/sbt project (partial)
```

A minimal Scala module in `build.mill` at the project root:

```scala
// build.mill
package build
import mill._, scalalib._

object foo extends ScalaModule {
  def scalaVersion = "3.4.2"
  def mvnDeps = Seq(mvn"com.lihaoyi::os-lib:0.10.0")

  object test extends ScalaTests {
    def mvnDeps = Seq(mvn"com.lihaoyi::utest:0.8.3")
    def testFramework = "utest.runner.Framework"
  }
}
```

```bash
./mill foo.compile          # compile sources
./mill foo.run              # run main
./mill foo.test             # run the nested test module
./mill show foo.assembly    # build an executable fat-jar, print its path
./mill resolve _            # list available tasks
```

## Architecture / How It Works

A Mill build is a Scala program that Mill compiles and caches as a **meta-build**
before running any of your tasks. The `./mill` launcher script is the only
checked-in binary dependency; it downloads the Mill version pinned in
`.mill-version` (or `.config/mill-version`), so builds are reproducible across
machines without a global install.

Tasks come in a few kinds, and the distinction matters for correctness:

- **Cached targets** (`def x = Task { ... }`) — memoized to `out/<module>/<x>.json`,
  with file outputs written under a per-task `.dest` scratch directory. Recomputed
  only when an upstream input hash changes.
- **Sources / inputs** (`Task.Source`, `Task.Input`) — the leaves that feed the
  graph; changes here are what trigger invalidation.
- **Commands** (`Task.Command`) — invocable from the CLI, never cached.
- **Workers** (`Task.Worker`) — long-lived in-memory state (e.g. the Zinc
  incremental compiler, kept warm across invocations).

Dependency resolution is done by coursier; incremental Scala/Java compilation by
Zinc. The task graph is executed in parallel by default, sized to available
cores. Task selectors are a small query language: `foo.compile`, wildcard
`foo._.test`, and cross-module axes like `foo[2.13.12].compile` for
cross-versioned builds. `inspect` and `show` let you introspect any node's
definition and its cached value.

The coupling worth understanding: because caching is keyed on **declared**
inputs, a task that reads a file or environment value it did not declare as a
`Task.Source`/`Task.Input` will produce a stale-cache bug that looks like
non-determinism. Correctness rests on declaring the full input surface — the
same discipline Bazel enforces, but here it is on the build author, not the tool.

## Production Notes

**Editing the build wants BSP.** `build.mill` is Scala, so IntelliJ or Metals
talking to Mill's Build Server Protocol server gives you completion and
navigation. Without it, editing a non-trivial build is guesswork; this is the
single biggest onboarding cost for teams new to Mill.

**Cross-version upgrades break the DSL.** Mill's API has changed meaningfully
across its 0.x line — the task macro moved from `T { ... }` to `Task { ... }`,
`T.dest` to `Task.dest`, and dependency helpers were renamed (`ivyDeps`/`ivy"..."`
to `mvnDeps`/`mvn"..."`). Pin `.mill-version`, upgrade deliberately, and expect
to edit build files on major bumps. The 0.x-to-1.0 transition consolidated much
of this.

**Plugin ecosystem is small.** Compared to Maven, Gradle, or even sbt, there are
far fewer off-the-shelf plugins. Mill's answer is that writing a custom task is
cheap because it is just a method — but "just write it yourself" is real work
when the incumbent tools ship a plugin.

**Migration is manual-ish.** `mill init` can import Maven, Gradle, and sbt
projects, but non-trivial builds (custom plugins, elaborate configuration) need
hand-translation. The `out/` cache directory also grows over time; `./mill clean`
resets it, and under-declared task inputs are the CI footgun to watch — a
"works on my machine" stale cache can hide there.

## When to Use / When Not

**Use when:**
- You run a JVM monorepo with many modules and want fast, parallel, incremental
  rebuilds without standing up Bazel.
- You build Scala, Scala.js, or Scala Native, where Mill's support is first-class.
- You want build logic that is real code you can factor, test, and step through.

**Avoid when:**
- The team will not touch a Scala-flavored DSL and your needs stay within
  Maven/Gradle conventions.
- You depend on a rich third-party plugin ecosystem (release automation, exotic
  packaging, vendor integrations) that already exists elsewhere.
- You are Android-first — Gradle is the supported and expected path there despite
  Mill's experimental Android support.

## Alternatives

- sbt/sbt — the incumbent Scala build tool; use it when you need the largest
  Scala plugin ecosystem and community-default project layouts.
- gradle/gradle — use for Android and when you want the biggest JVM plugin
  ecosystem plus a mature Kotlin DSL.
- apache/maven — use when maximum stability and universal CI/IDE support matter
  more than build speed or flexibility.
- bazelbuild/bazel — use for very large polyglot monorepos that need hermetic
  builds and remote caching across a fleet.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017–2018 | Initial public release; programmable Scala builds by Li Haoyi[^1]. |
| 0.11 | 2023 | Large API cleanup across the task/module surface. |
| 0.12 | 2024 | Continued 0.x refinement ahead of the 1.0 line. |
| 1.0 | 2025 | Stable API; Java/Kotlin first-class, declarative `build.mill.yaml`[^3]. |
| 1.1.7 | 2026 | Current stable release on Maven Central[^2]. |

## References

[^1]: Li Haoyi, "Mill: A Build Tool based on Pure Functional Programming."
https://www.lihaoyi.com/post/MillBetterScalaBuilds.html
[^2]: Mill README and Maven Central `com.lihaoyi:mill-dist` (stable 1.1.7).
https://github.com/com-lihaoyi/mill
[^3]: Mill documentation — build configuration and comparisons.
https://mill-build.org/

## Tags

scala, java, kotlin, build-tool, build-system, jvm, incremental-build, caching, monorepo, task-graph
