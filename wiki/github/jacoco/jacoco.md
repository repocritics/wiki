# jacoco/jacoco

> Java code coverage measured by bytecode instrumentation — the engine behind almost every "coverage %" number in the JVM ecosystem.

[GitHub repo](https://github.com/jacoco/jacoco) ·
[Official website](https://www.jacoco.org/jacoco) ·
[License: EPL-2.0](https://github.com/jacoco/jacoco/blob/master/LICENSE.md)

## Overview

JaCoCo (Java Code Coverage) is a free coverage library for the JVM, maintained under the Eclipse Foundation umbrella by Mountainminds / the EclEmma team[^1]. It is the de facto standard: when Maven, Gradle, SonarQube, or a CI dashboard reports a coverage percentage for Java, Kotlin, Groovy, or Scala code, JaCoCo almost certainly produced it. It succeeded EMMA (unmaintained since ~2005) and Cobertura as the default coverage tool for the JVM.

The library measures coverage by instrumenting Java bytecode, not source. This is its defining tradeoff. Working at the bytecode level means JaCoCo needs no source access at runtime, attaches to any JVM as a Java agent, and covers any language that compiles to class files. But it also means coverage is reported in terms of what the compiler actually emitted — synthetic methods, generated accessors, bridge methods, null checks, and inlined code all appear as bytecode with their own branches, which is why raw JaCoCo numbers for Kotlin or Lombok-heavy code can look wrong until filtering is understood[^2].

Notably, JaCoCo has never shipped a 1.0. It has lived in the `0.8.x` line for years, with each release primarily adding support for the next Java class file version and refining its filtering framework. It is actively maintained (recent commits, ~4.6k stars, ~1.2k forks) but deliberately conservative in scope: it measures coverage and generates reports, nothing more.

## Getting Started

Most projects never invoke JaCoCo directly — they use the Maven or Gradle plugin.

```xml
<!-- Maven: bind the agent to test, generate an HTML/XML report -->
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.12</version>
  <executions>
    <execution><goals><goal>prepare-agent</goal></goals></execution>
    <execution>
      <id>report</id><phase>test</phase>
      <goals><goal>report</goal></goals>
    </execution>
  </executions>
</plugin>
```

```groovy
// Gradle: the jacoco plugin ships with Gradle itself
plugins { id 'jacoco' }
test { finalizedBy jacocoTestReport }
jacocoTestReport { reports { xml.required = true; html.required = true } }
```

```bash
# Raw agent — attach to any JVM, dump an .exec file on exit
java -javaagent:jacocoagent.jar=destfile=jacoco.exec,append=false -jar app.jar
# Then turn the binary .exec into a report with the CLI:
java -jar jacococli.jar report jacoco.exec \
  --classfiles target/classes --sourcefiles src/main/java --html report/
```

## Architecture / How It Works

JaCoCo has three moving parts: the **agent** that records execution, the binary **`.exec`** file it produces, and the **report generator** that joins execution data back to class files.

- **On-the-fly instrumentation (default).** The agent uses `java.lang.instrument` to transform classes as they are loaded, inserting probes that flip boolean flags when a code path executes. Probe placement is derived from control-flow analysis of the bytecode, built on the ASM library[^3]. Because it hooks classloading, it needs no build-time step.
- **Offline instrumentation.** For environments where an agent can't attach or where classes are already transformed by another agent (Android's build uses this; PowerMock and other bytecode rewriters can conflict), JaCoCo instruments class files ahead of time and uses a small runtime to collect data[^4].
- **Counters.** JaCoCo reports several dimensions from a single run: *instructions* (its most granular, bytecode-level C0), *branches* (C1, decision coverage), *lines* (derived from debug line tables), *methods*, *classes*, and *cyclomatic complexity*. Line coverage requires the classes to be compiled with debug information; without it, only instruction and branch coverage are available.
- **Filtering framework (since 0.8.0).** A pipeline of filters removes bytecode patterns that shouldn't count against you: synthetic/bridge methods, `try-with-resources` desugaring, `assert` statements, Kotlin `when`/coroutine/inline artifacts, Lombok and other `@Generated`-annotated code[^2]. This layer is where most of JaCoCo's per-release effort goes, and why Kotlin coverage keeps improving version to version.

The report generator is a separate concern: it reads one or more `.exec` files plus the original class files (and optionally sources) and emits HTML, XML, or CSV. The XML format is the integration contract consumed by SonarQube, Codecov, Coveralls, and CI plugins.

## Production Notes

**New JDK support is the recurring pain.** JaCoCo can only instrument class file versions its bundled ASM understands. Run tests on a JDK newer than your JaCoCo release supports and instrumentation fails outright — typically `Unsupported class file major version NN` or `Error while instrumenting`. Every new Java release triggers a wave of broken builds until teams bump JaCoCo. This is the single most common JaCoCo incident, and the reason it should be one of the first dependencies you upgrade when adopting a new JDK[^5].

**Kotlin and Lombok numbers need filtering context.** Inline functions, `data class` generated methods, null-safety checks, and coroutine state machines all emit extra bytecode and synthetic branches. Even with the filtering framework, branch coverage on Kotlin can read lower than a source-level tool would report. Treat absolute Kotlin branch percentages with skepticism and compare trends, not thresholds copied from Java projects.

**Forked JVMs and multi-module merges.** The agent only records the JVM it is attached to. Integration tests that fork processes, Surefire/Failsafe `forkCount > 1`, or separately-run integration suites each produce their own `.exec` file. Aggregating unit + integration coverage means running the agent in `append` mode or merging exec files (`jacoco:merge` / CLI `merge`) and pointing the report at all class directories. Missing this is why "our integration tests run but coverage shows 0%" reports are common.

**Coverage checks can gate builds.** The `check` goal (Maven) / `jacocoTestCoverageVerification` (Gradle) fails the build when a counter drops below a rule. Useful, but rules are per-counter and per-scope; a naive `INSTRUCTION >= 0.80` rule behaves very differently from a `BRANCH`/`LINE` rule, and module-level vs bundle-level scope trips teams up.

**Instrumentation overhead is modest but real** — extra memory for probe arrays and some runtime cost. Rarely a problem for test suites; occasionally noticeable when profiling coverage on very large long-running applications via `tcpserver` output mode.

## When to Use / When Not

**Use when:**
- You need line/branch/instruction coverage for any JVM language and want the ecosystem-standard XML that SonarQube, Codecov, and CI dashboards already understand.
- You want zero build-time coupling — attach an agent to a running JVM and collect coverage from integration or system tests.
- You're on Maven or Gradle and want coverage with a few lines of config.

**Avoid / look elsewhere when:**
- You want *mutation* testing (does your test actually assert anything?) — JaCoCo only measures execution, not test quality. Use PIT.
- You need source-accurate coverage for a language whose bytecode diverges heavily from source and filtering isn't enough for your reporting needs.
- You're pinned to a bleeding-edge JDK preview that no JaCoCo release supports yet — you may have to wait for a compatible version.

## Alternatives

- pitest/pitest — mutation testing; measures whether tests *detect* faults, not just execute lines. Complements rather than replaces JaCoCo.
- cobertura/cobertura — the older JVM coverage tool JaCoCo displaced; effectively unmaintained, avoid for new work.
- Use IntelliJ IDEA's built-in coverage runner when you only need coverage locally in the IDE and don't require a CI-consumable report.
- Use a language-native tool (kotlinx-kover wraps JaCoCo/IntelliJ engines) when you want Kotlin-first ergonomics over raw JaCoCo config.
- Use SonarQube on top of JaCoCo when you want coverage tracked alongside quality gates and history rather than a one-shot report.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.5.x | 2011 | Early public releases; EMMA successor via the EclEmma project[^1]. |
| 0.7.x | 2014–2017 | Broad Maven/Gradle adoption; Java 8 support. |
| 0.8.0 | 2018-01 | Filtering framework introduced (synthetic code, try-with-resources, etc.)[^2]. |
| 0.8.7 | 2021 | Java 17 support. |
| 0.8.8 | 2022 | Java 18 support. |
| 0.8.11 | 2023 | Java 21 support. |
| 0.8.12 | 2024 | Java 22/23 class file support; continued Kotlin filtering work[^6]. |

## References

[^1]: JaCoCo project home and mission. https://www.jacoco.org/jacoco/
[^2]: JaCoCo documentation, "Filtering Options" / changelog for filtering framework. https://www.jacoco.org/jacoco/trunk/doc/changes.html
[^3]: ASM bytecode framework, used by JaCoCo for instrumentation. https://asm.ow2.io/
[^4]: JaCoCo documentation, "Offline Instrumentation." https://www.jacoco.org/jacoco/trunk/doc/offline.html
[^5]: JaCoCo release notes track supported class file versions per release. https://github.com/jacoco/jacoco/releases
[^6]: JaCoCo change log. https://www.jacoco.org/jacoco/trunk/doc/changes.html

## Tags

java, kotlin, code-coverage, testing, jvm, bytecode-instrumentation, java-agent, maven, gradle, asm, static-analysis
