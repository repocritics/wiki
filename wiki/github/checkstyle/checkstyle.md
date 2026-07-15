# checkstyle/checkstyle

> AST-based static analysis for Java that enforces coding-standard conformance — style, conventions, and structure, not runtime bugs.

[GitHub repo](https://github.com/checkstyle/checkstyle) ·
[Official website](https://checkstyle.org) ·
[License: LGPL-2.1](https://github.com/checkstyle/checkstyle/blob/master/LICENSE)

## Overview

Checkstyle is a source-level linter for Java. It parses `.java` files into an
abstract syntax tree and runs a configurable set of "check" modules over that
tree, emitting violations when code deviates from a declared coding standard. It
was created by Oliver Burn around 2001 and lived on SourceForge for years before
the canonical repository moved to GitHub in 2013[^1]. It is one of the oldest
still-active tools in the Java quality ecosystem and remains the de facto answer
to "how do we enforce a house style in CI."

The defining characteristic is that Checkstyle checks *conventions*, not
correctness. It will flag a missing Javadoc comment, an import ordering problem,
a method that is too long, a brace on the wrong line, or a magic number — but it
does not find null-dereferences, resource leaks, or concurrency bugs. That is the
territory of SpotBugs (bytecode) and PMD/Error Prone (different AST rule sets).
Teams routinely run Checkstyle alongside those tools rather than instead of them.

The persistent tension is configuration cost versus value. Checkstyle ships two
reference configs — `google_checks.xml` and `sun_checks.xml`[^2] — but any real
project ends up maintaining a bespoke XML config. That XML is verbose,
order-sensitive in places, and its semantics shift between major versions, so an
upgrade can surface a wave of new violations on unchanged code.

## Getting Started

Standalone CLI (requires a JRE; Checkstyle 10.x needs Java 11+[^3]):

```bash
# Grab the all-in-one jar from the GitHub Releases page, then:
java -jar checkstyle-10.x.x-all.jar -c /google_checks.xml MyClass.java
```

Typical usage is through a build plugin. Maven:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-checkstyle-plugin</artifactId>
  <version>3.3.1</version>
  <configuration>
    <configLocation>checkstyle.xml</configLocation>
    <failOnViolation>true</failOnViolation>
  </configuration>
</plugin>
```

A minimal `checkstyle.xml`:

```xml
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
  "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
  "https://checkstyle.org/dtds/configuration_1_3.dtd">
<module name="Checker">
  <property name="severity" value="warning"/>
  <module name="LineLength">
    <property name="max" value="120"/>
  </module>
  <module name="TreeWalker">
    <module name="UnusedImports"/>
    <module name="MethodLength"/>
    <module name="MagicNumber"/>
  </module>
</module>
```

## Architecture / How It Works

Checkstyle has a two-level module tree rooted at a `Checker`. Modules that need
Java syntax live inside a `TreeWalker`; modules that operate on raw text or
whole files sit directly under `Checker`.

- **TreeWalker checks** run against an AST. Checkstyle generates its Java parser
  from an ANTLR grammar; each source file is tokenized and parsed into a tree of
  `DetailAST` nodes, and each check registers for the token types it cares about
  (e.g. `METHOD_DEF`, `IMPORT`). This is why Checkstyle can reason about scope,
  modifiers, and Javadoc structure that a regex never could.
- **File/line checks** (`LineLength`, `RegexpHeader`, `FileLength`,
  `NewlineAtEndOfFile`) never see the AST — they run over lines of text, so they
  also work on files the parser can't fully understand.

Because the parser is grammar-driven, Checkstyle can only analyze language
syntax its grammar version understands. New Java features (records, sealed
classes, pattern matching, virtual-thread syntax) require a grammar update; on
an older Checkstyle against newer source, `TreeWalker` checks throw parse errors
rather than silently skipping. Keeping Checkstyle current with the JDK you
compile against is a hard requirement, not a nicety.

Suppression is layered: inline `// CHECKSTYLE:OFF` comment filters, a
`SuppressionFilter` pointing at a `suppressions.xml` of file/check regexes,
`SuppressWarningsFilter` honoring `@SuppressWarnings("checkstyle:...")`
annotations, and per-module `files`/`fileExtensions` properties. Precedence and
the requirement to explicitly wire in the filter modules trip up most first-time
configs.

## Production Notes

- **Version upgrades are the main operational cost.** New checks are added,
  default property values change, and previously lenient behavior tightens across
  minor releases. A bump from, say, 9.x to 10.x can turn a green build red with
  no source change. Pin the exact version in the build and upgrade deliberately,
  reviewing the release notes' "breaking" section each time.
- **Java version coupling is real.** Checkstyle 10.x requires Java 11 to *run*[^3],
  independent of the bytecode target of the project you analyze. Analyzing code
  that uses syntax newer than your Checkstyle grammar produces `NoViableAltException`
  parse failures, not graceful warnings.
- **Style ≠ formatting.** Checkstyle reports violations; it does not rewrite code.
  Enforcing a format with Checkstyle and expecting developers to hand-fix is a
  friction generator. Most teams pair it with an actual formatter
  (google-java-format via Spotless) so the machine fixes layout and Checkstyle
  guards the rules a formatter can't express (Javadoc presence, naming, design).
- **The Maven and Gradle plugins version-pin Checkstyle independently.** The
  `maven-checkstyle-plugin` bundles a default Checkstyle version that lags the
  standalone release; override `<dependency>` on `checkstyle` inside the plugin to
  control which engine actually runs, or config/engine mismatch causes confusing
  "unknown module" errors.
- **Performance is fine but not free.** Parsing every file has cost on large
  monorepos; Checkstyle caches results between runs via a `cacheFile` so unchanged
  files are skipped. Ensure the cache is preserved between CI runs to avoid
  full re-analysis each build.
- **Config drift between IDE and CI** is common: developers see a clean build
  locally (IDE plugin on a different config/version) then CI fails. Point both at
  the same `checkstyle.xml` and pin the same engine version.

## When to Use / When Not

**Use when:**
- You need to enforce a consistent house style / conventions across a Java team in CI.
- You want rule categories a formatter can't express: Javadoc completeness, naming
  conventions, import restrictions, design constraints, complexity limits.
- You're adopting Google or Sun style and want a ready-made reference config.

**Avoid (or supplement) when:**
- You want to *find bugs* — use SpotBugs, PMD, or Error Prone instead/alongside.
- You want code auto-formatted, not just reported — use a formatter (Spotless +
  google-java-format).
- Your codebase is polyglot — Checkstyle is Java-only; reach for a multi-language
  linter or SonarQube.

## Alternatives

- pmd/pmd — AST-based, overlaps on style but leans toward code smells, complexity, and copy-paste detection; use when you want rule breadth beyond conventions.
- spotbugs/spotbugs — analyzes compiled bytecode for likely bugs (null deref, bad casts); use for correctness defects Checkstyle structurally cannot see.
- google/error-prone — compile-time checks wired into javac with auto-fixes; use when you want bug-pattern enforcement inside the build with suggested fixes.
- diffplug/spotless — formatting orchestrator wrapping google-java-format et al.; use when the goal is to *fix* layout rather than *report* it.
- SonarSource/sonarqube — dashboard-driven multi-language quality platform; use when you need cross-language analysis, historical trends, and quality gates over a single-language linter.

## History

| Version | Date | Notes |
|---------|------|-------|
| ~1.0 | ~2001 | Initial release on SourceForge by Oliver Burn[^1]. |
| — | 2013-08-31 | Repository established on GitHub[^1]. |
| 6.0 | 2014 | Major cleanup era; API modernization. |
| 8.0 | 2017 | Requires Java 8; large check additions over the 8.x line. |
| 10.0 | 2022 | Drops Java 8, requires Java 11 to run[^3]; ongoing grammar updates for new JDK syntax. |

## References

[^1]: checkstyle/checkstyle repository — created 2013-08-31; project predates GitHub on SourceForge. https://github.com/checkstyle/checkstyle
[^2]: Checkstyle bundled configurations (`google_checks.xml`, `sun_checks.xml`). https://checkstyle.org/style_configs.html
[^3]: Checkstyle documentation — requirements and running. https://checkstyle.org/cmdline.html

## Tags

java, static-analysis, linter, code-quality, coding-standard, ast, jvm, ci, style-checker, code-review
