# google/error-prone

> A static analysis tool that plugs into javac to catch common Java mistakes as compile-time errors.

[GitHub repo](https://github.com/google/error-prone) ·
[Official website](https://errorprone.info) ·
[License: Apache-2.0](https://github.com/google/error-prone/blob/master/COPYING)

## Overview

Error Prone is a Java static analysis tool built and maintained by Google that runs
*inside the compiler*. Rather than scanning bytecode or source as a separate pass,
it hooks into `javac` and inspects the same abstract syntax tree the compiler
builds, then emits its findings as ordinary compiler diagnostics — warnings or
hard errors that fail the build[^1]. The GitHub repository dates to 2014, but the
project originated earlier as an internal Google tool and has been open source
since the early 2010s[^2]. At ~7.2k stars and a release roughly every six weeks
(v2.50.0 shipped 2026-06), it is actively developed and used at very large scale
inside Google itself.

The defining bet is that the *compiler* is the right place to catch bugs: findings
appear in the same output as type errors, with the same line-precise location, and
a check promoted to `ERROR` severity is impossible to ignore because the build
stops. The cost of that bet is deep coupling to `javac` internals — Error Prone
reads packages like `com.sun.tools.javac.*` that the JDK does not consider public
API. That single design decision drives most of the tool's operational friction
(see Production Notes).

Error Prone ships two things people conflate. The **checker** is the analysis
engine plus a few hundred built-in bug patterns. The **`error_prone_annotations`**
artifact is a tiny, dependency-free jar of annotations (`@CheckReturnValue`,
`@CanIgnoreReturnValue`, `@Immutable`, `@FormatMethod`, `@CompileTimeConstant`,
`@Var`, `@RestrictedApi`) that libraries such as Guava depend on to drive checks in
their consumers. You can depend on the annotations without ever running the checker.

## Getting Started

Configure it as a `javac` plugin via your build tool. Maven example:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <compilerArgs>
      <arg>-XDcompilePolicy=simple</arg>
      <arg>--should-stop=ifError=FLOW</arg>
      <arg>-Xplugin:ErrorProne -Xep:CollectionIncompatibleType:ERROR</arg>
    </compilerArgs>
    <annotationProcessorPaths>
      <path>
        <groupId>com.google.errorprone</groupId>
        <artifactId>error_prone_core</artifactId>
        <version>2.50.0</version>
      </path>
    </annotationProcessorPaths>
  </configuration>
</plugin>
```

A minimal example of what it catches — a type-mismatched collection call that
`javac` alone accepts because `Collection.remove` takes `Object`:

```java
Set<Short> s = new HashSet<>();
s.remove(i - 1);   // i is short, i - 1 is int
// error: [CollectionIncompatibleType] Argument 'i - 1' should not be passed to
// this method; its type int is not compatible with its collection's type Short
```

Severity of any check is tuned per build with `-Xep:CheckName:OFF|WARN|ERROR`.

## Architecture / How It Works

Each check is a `BugChecker` subclass annotated with `@BugPattern` (name, summary,
default severity). It implements one or more `*TreeMatcher` interfaces
(`MethodInvocationTreeMatcher`, `ReturnTreeMatcher`, …); Error Prone walks the
compilation unit and dispatches each AST node to the matchers that care about it.
Matchers are composed from a `Matcher<T>` combinator library, and checks can query
resolved types and symbols because they run *after* `javac`'s attribution phase —
this is what lets it reason about generics and overload resolution rather than just
syntax.

A check can attach a `SuggestedFix` (an edit against source text). With
`-XepPatchChecks:...` and `-XepPatchLocation:...`, Error Prone runs as a batch
refactoring tool, writing the fixes back to disk instead of merely reporting them —
the mechanism used for large-scale automated cleanups.

**Refaster** is a sibling capability: you express a rewrite as a `@BeforeTemplate`
method and an `@AfterTemplate` method, and Refaster compiles those into an AST
pattern-and-replace. It is how many idiomatic-migration rules are written without
hand-coding a `BugChecker`.

Because checks execute in the compiler, they depend on `error_prone_check_api`,
which reaches into `com.sun.tools.javac` internals. The maintainers state plainly
that this check API is **not** stable across releases — custom checks routinely
need adjustment when you upgrade[^3]. This is the coupling story in one sentence:
power comes from sharing `javac`'s data structures, and the price is versioned
against them.

## Production Notes

**JDK module access is the number-one footgun.** Since JDK 16 strongly encapsulated
internal packages (JEP 396), running Error Prone requires passing a block of
`--add-exports jdk.compiler/com.sun.tools.javac.*=ALL-UNNAMED` and `--add-opens`
flags to the compiler JVM[^4]. Miss one and you get an opaque
`IllegalAccessError` at compile time, not a clear message. Bazel's Java toolchain
handles this for you; Maven and Gradle users must copy the flag list (into
`.mvn/jvm.config` or the Gradle `forkOptions`) and keep it in sync across JDK
upgrades.

**It runs on every compile.** Error Prone adds real wall-clock overhead
proportional to the number of enabled checks, and it forks the compiler, so it does
not benefit from a warm in-process `javac`. Teams often run the full check set in
CI but a reduced set (or none) for local incremental builds.

**Adopt severities gradually.** Turning on a large check set at `ERROR` against an
existing codebase will flood the build with failures. The standard rollout is to
introduce new checks at `WARN`, burn down the findings (the batch patcher helps),
then promote to `ERROR` so they cannot regress. Individual sites opt out with
`@SuppressWarnings("CheckName")`.

**JDK version tracking.** Because it binds to compiler internals, Error Prone
supports a moving window of JDKs, and support for a brand-new JDK release can lag.
Newer releases have also raised the minimum JDK required to *run* the compiler even
though `--release` can still target older bytecode. Pin the Error Prone version to
one known-good with your JDK rather than floating it.

**IDE gap.** Error Prone lives in the compiler, not the editor. Findings surface in
build output and CI, not natively as you type; IDE integration is limited compared
to editor-native linters. Treat it as a build gate, not an interactive assistant.

## When to Use / When Not

**Use when:**
- You have a Java codebase and want bug classes enforced as build failures, not advisory reports.
- You depend on libraries (Guava, others) that ship `@CheckReturnValue` / `@CanIgnoreReturnValue` and want those contracts enforced.
- You want to write org-specific checks or Refaster rules to codify local conventions and ban APIs (`@RestrictedApi`).
- You are on Bazel, where setup cost is near zero.

**Avoid when:**
- You are not on Java, or you are stuck on an old/unusual JDK the current releases no longer run on.
- You cannot tolerate the `javac` module-flag configuration burden or the per-compile overhead in your build.
- You want editor-time, as-you-type feedback more than a build gate — a native IDE inspection set fits better.

## Alternatives

- pmd/pmd — rule-based source analysis that runs as a standalone pass; broader style/rule coverage, but not wired into the type-resolved compiler AST. Use when you want language-agnostic, configurable rulesets outside the compile step.
- spotbugs/spotbugs — analyzes compiled bytecode (the maintained successor to FindBugs). Use when you want post-compile analysis independent of the source build and its own bug catalog.
- checkstyle/checkstyle — enforces formatting and structural conventions, not semantic bug detection. Use it alongside, not instead of, Error Prone.
- uber/NullAway — a null-safety analyzer built *as* an Error Prone plugin. Use it with Error Prone when your primary concern is NullPointerException elimination.
- facebook/infer — interprocedural analysis (null, resource leaks, concurrency) across multiple languages. Use when you need whole-program reasoning beyond single-compilation-unit checks.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2014-08-21 | GitHub repo; project originated earlier inside Google[^2]. |
| 2.3.0 | 2018-04-19 | Mature 2.x line; Refaster and batch patching established. |
| 2.4.0 | 2020-05-29 | Continued 2.x cadence. |
| 2.5.0 | 2021-01-12 | JDK 16 era begins — module `--add-exports` flags become required to run[^4]. |
| 2.48.0 | 2026-02-27 | Recent release. |
| 2.49.0 | 2026-04-07 | Recent release. |
| 2.50.0 | 2026-06-10 | Current release at time of writing; ~six-week cadence. |

## References

[^1]: Error Prone documentation — home. https://errorprone.info
[^2]: google/error-prone repository (created 2014-08-21) and project background. https://github.com/google/error-prone
[^3]: Error Prone — "Criteria for a new check" / plugin check notes on API stability. https://errorprone.info/docs/plugins
[^4]: Error Prone — installation and required `--add-exports`/`--add-opens` flags for JDK 16+. https://errorprone.info/docs/installation

## Tags

java, static-analysis, javac, compiler-plugin, linter, bug-detection, code-quality, refactoring, google, jvm
