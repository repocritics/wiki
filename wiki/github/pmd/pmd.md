# pmd/pmd

> An extensible multilanguage static source-code analyzer that runs rules against parsed ASTs — best known for Java and Apex.

[GitHub repo](https://github.com/pmd/pmd) ·
[Official website](https://pmd.github.io) ·
[License: BSD-style](https://github.com/pmd/pmd/blob/main/LICENSE)

## Overview

PMD is a static source-code analyzer that parses code into an abstract syntax tree (AST) and evaluates rules against that tree to flag likely defects and style problems: unused variables, empty catch blocks, unnecessary object creation, overly complex methods, and so on[^1]. Unlike bytecode analyzers, it works on source and does not require a compiled artifact to run (though type-aware Java rules benefit from one — see Production Notes). Its center of gravity is Java and Salesforce Apex, but it ships parsers and/or rules for roughly a dozen other languages, and bundles CPD, a separate copy-paste detector that operates on a much wider language set.

The project predates its GitHub history: it began on SourceForge in the early 2000s, which is why published artifacts still carry the Maven groupId `net.sourceforge.pmd`. The GitHub repository was created in 2012[^2]; the codebase itself is considerably older. This long lineage shows up in the rule catalog, which accumulated across major versions and was pruned and re-organized in the 7.0 rewrite.

PMD's defining tension is signal versus noise. It ships 400+ built-in rules, and a large fraction of them are opinionated style checks that will produce false positives on any real codebase until tuned. Teams that adopt it wholesale and treat every finding as a build failure usually abandon it; teams that curate a small ruleset and wire in suppression conventions tend to keep it for years. It is a linting framework you configure, not a turnkey bug oracle.

## Getting Started

Download the binary distribution from the releases page and unzip, or use the Docker image `pmdcode/pmd`. Maven and Gradle plugins exist for build integration. The PMD 7 CLI is subcommand-based:

```bash
# Analyze a source tree with the bundled Java "quickstart" ruleset
pmd check -d src/main/java -R rulesets/java/quickstart.xml -f text

# Copy-paste detection, reporting duplicate blocks of >= 100 tokens
pmd cpd --minimum-tokens 100 --dir src
```

A ruleset is an XML file that references categories or individual rules:

```xml
<?xml version="1.0"?>
<ruleset name="Custom Rules"
         xmlns="http://pmd.sourceforge.net/ruleset/2.0.0">
  <description>Curated subset for CI</description>

  <!-- pull in one whole category -->
  <rule ref="category/java/errorprone.xml"/>

  <!-- or a single rule, with a tuned property -->
  <rule ref="category/java/design.xml/CyclomaticComplexity">
    <properties>
      <property name="methodReportLevel" value="15"/>
    </properties>
  </rule>
</ruleset>
```

## Architecture / How It Works

PMD's pipeline is: **tokenize/parse → build AST → run rules → collect violations → render report.** Parsers are per-language; historically most were generated with JavaCC, with Antlr used for several newer language modules[^1]. Each language module contributes a parser, a node hierarchy, and its rule set.

Rules come in two forms:

1. **Java rules** — a class (typically a visitor over the language's node types) that walks the AST and reports violations. These can do type resolution, symbol-table lookups, and multi-node reasoning.
2. **XPath rules** — an XPath expression evaluated against the AST, where AST nodes are addressed like XML elements and node attributes expose properties. This is the lower-effort path and the one the rule designer targets. PMD 7 standardized on XPath 3.1 (evaluated via Saxon); the older XPath 1.0 engine was dropped.

Rules are grouped into **categories** — `bestpractices`, `codestyle`, `design`, `documentation`, `errorprone`, `multithreading`, `performance`, `security` — which is how the `category/<lang>/<category>.xml` references in rulesets resolve.

**CPD** is architecturally separate. It tokenizes source (a much simpler front end than the full parsers, which is why it supports ~35 languages including C/C++, Python, Go, Ruby, and TypeScript that have no full PMD rule support) and finds the longest matching token sequences across files. The `--minimum-tokens` threshold is the primary tuning knob: too low and you drown in boilerplate matches, too high and you miss real duplication.

The 7.0 line was a ground-up API rewrite: a unified AST/tree API across languages, a reworked language-module registration system, a new CLI, and removal of a large batch of deprecated rules and Java internals. This is why custom Java rules written against PMD 6 generally do not compile unchanged against 7[^3].

## Production Notes

**Auxclasspath is not optional for serious Java analysis.** Type-resolution rules (anything that reasons about the actual types of expressions, method resolution, or class hierarchies) degrade silently when PMD cannot load the project's compiled classes and dependencies. Pass the project classpath via `--aux-classpath` (or let the Maven/Gradle plugin do it). Without it you get more false positives and missed findings, with no hard error to tell you why.

**Incremental analysis cache.** Full runs on large monorepos are slow. The `--cache <file>` option persists results so unchanged files are skipped on the next run; on CI this is frequently a 5–10x wall-clock difference. The cache must be keyed to a stable location and invalidated when the ruleset or PMD version changes (PMD handles the latter).

**Suppression is a first-class workflow, plan for it.** Real projects need escape hatches: `@SuppressWarnings("PMD")` or `@SuppressWarnings("PMD.RuleName")` on Java elements, a `// NOPMD` line comment, or XPath/regex-based suppression configured in the ruleset. Adopting PMD without agreeing on a suppression convention leads to either a wall of ignored warnings or churn from developers disabling rules wholesale.

**False positives are the norm, not the exception.** The bundled `quickstart` ruleset is deliberately conservative and is the recommended starting point over "enable everything." Expect to spend real time curating the active set; treating the full 400+ catalog as a quality gate is the most common way teams come to resent the tool.

**PMD 6 → 7 is a genuine migration, not a version bump.** Rules were renamed, moved between categories, or removed; the ruleset XML is mostly compatible but references to deleted/renamed rules fail; the CLI changed from flag-style (`pmd -d ... -R ...`) to subcommands (`pmd check -d ... -R ...`), breaking scripts and CI invocations; and custom Java rules need porting to the new API. Budget for it.

**Runtime.** PMD is a JVM application distributed as a self-contained zip; it needs a Java runtime present on the machine that runs the analysis, independent of what language you are analyzing.

## When to Use / When Not

**Use when:**
- You analyze Java or Apex and want AST-level, source-based rules you can read, tune, and extend.
- You need custom rules expressed as XPath or Java visitors without building your own analyzer.
- You want cross-language copy-paste detection (CPD) across a polyglot repo.
- You want a rule engine that runs offline in CI with no server component.

**Avoid when:**
- You want bytecode/dataflow bug detection (null-deref, resource leaks) — a bytecode analyzer catches classes of bugs PMD's source AST rules do not.
- You want zero-configuration, low-false-positive checks out of the box — expect a tuning investment.
- Your primary language has full PMD parser/rule support only partially (many CPD-only languages have no rule engine — you get duplication detection but not lint rules).
- You want in-compiler checks with autofixes wired into the build itself.

## Alternatives

- spotbugs/spotbugs — analyzes JVM bytecode instead of source; use it when you want dataflow-style bug patterns (null derefs, bad casts) that source-AST rules miss.
- checkstyle/checkstyle — formatting and coding-convention enforcement; use it when your concern is style/layout consistency rather than logic and design smells.
- google/error-prone — a javac plugin that flags bugs at compile time, often with suggested fixes; use it when you want checks fused into the build rather than a separate pass.
- facebook/infer — interprocedural analysis for null-safety, resource, and concurrency issues across Java/C/Objective-C; use it for deeper semantic bugs than a per-file linter finds.
- SonarSource/sonarqube — a server platform with dashboards and quality gates (its own analyzers); use it when you want tracked trends and gating rather than a CLI in CI.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | ~2002 | Origin as a SourceForge project (groupId `net.sourceforge.pmd`)[^2]. |
| 5.0.0 | 2012 | Major line; broadened language and rule support. |
| 6.0.0 | 2018 | Rules reorganized into categories; incremental analysis cache; long-lived 6.x series. |
| 7.0.0 | 2024-03 | Ground-up API/AST rewrite, new CLI subcommands, XPath 3.1 via Saxon, large rule pruning[^3]. |

## References

[^1]: PMD README and project documentation — description, supported languages, JavaCC/Antlr parsing, CPD. https://github.com/pmd/pmd and https://docs.pmd-code.org/latest/
[^2]: GitHub repository metadata (repo created 2012-07-11; `net.sourceforge.pmd` groupId reflects the older SourceForge origin). https://github.com/pmd/pmd
[^3]: PMD 7 migration guide — API rewrite, rule/CLI changes. https://docs.pmd-code.org/latest/pmd_userdocs_migrating_to_pmd7.html

## Tags

java, apex, static-analysis, static-code-analysis, linter, code-quality, ast, copy-paste-detection, xpath, jvm, code-analysis
