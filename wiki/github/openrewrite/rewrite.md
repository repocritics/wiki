# openrewrite/rewrite

> Type-aware, rule-driven mass refactoring for the JVM — codemods that understand your code instead of pattern-matching text.

[GitHub repo](https://github.com/openrewrite/rewrite) ·
[Official website](https://docs.openrewrite.org) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

OpenRewrite is an automated source-refactoring engine originally built at Netflix by Jonathan Schneider (also the author of Micrometer) and now stewarded by Moderne, the commercial company he co-founded[^1]. It applies prepackaged "recipes" — framework migrations, security fixes, and style normalizations — to a codebase and rewrites the source in place, aiming to turn changes that would take hours or days of manual edits into a single command.

Its defining idea is the **Lossless Semantic Tree (LST)**: rather than operate on plain text or a throwaway AST, OpenRewrite parses source into a type-attributed tree that also preserves whitespace, comments, and formatting, so a transformation can reason about types *and* print back byte-identical unchanged regions[^2]. This is what separates it from `sed`-style codemods — a recipe can know that `List` resolves to `java.util.List` and not touch an unrelated `List` from another package. It is also the source of its main cost: building an LST requires resolving the project's full dependency classpath, which is slow and memory-hungry.

The project sits at an open-core boundary that shapes everything around it. The parsers, the LST, and a large catalog of base recipes are Apache-2.0 and always will be, per Moderne[^3]. But running recipes across many repositories at once — the batch, serialize-once model — is the commercial Moderne platform, and some recipes in the wider catalog carry Moderne licensing. Read the framework as the single-repo, open-source substrate; read Moderne as the multi-repo product built on top.

## Getting Started

Run a single recipe against one repository with the Maven or Gradle plugin. No code required for catalog recipes:

```bash
# Maven — dry-run first (writes a patch under target/), then apply
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.activeRecipes=org.openrewrite.java.format.AutoFormat \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite:rewrite-java
```

```groovy
// build.gradle — declarative activation
plugins { id("org.openrewrite.rewrite") version("latest.release") }
rewrite { activeRecipe("org.openrewrite.java.migrate.UpgradeToJava21") }
dependencies { rewrite("org.openrewrite.recipe:rewrite-migrate-java:latest.release") }
// then:  ./gradlew rewriteDryRun   (preview)   /   ./gradlew rewriteRun (apply)
```

A custom recipe is a visitor over the LST:

```java
public class SayHelloRecipe extends Recipe {
    @Override public String getDisplayName() { return "Add hello() method"; }
    @Override public String getDescription() { return "Adds a no-op hello() to a class."; }
    @Override public TreeVisitor<?, ExecutionContext> getVisitor() {
        return new JavaIsoVisitor<>() {
            @Override public J.ClassDeclaration visitClassDeclaration(
                    J.ClassDeclaration cd, ExecutionContext ctx) {
                // inspect cd.getType(), synthesize methods via JavaTemplate, etc.
                return super.visitClassDeclaration(cd, ctx);
            }
        };
    }
}
```

## Architecture / How It Works

The pipeline is **parse → LST → visit → print**. A language parser (`rewrite-java`, `rewrite-kotlin`, `rewrite-groovy`, and others) reads source and produces an LST whose nodes carry type attribution. Recipes contribute a `TreeVisitor`; the engine walks the tree, visitors return modified nodes, and an unchanged subtree is printed back exactly as it came in — trailing spaces, comments, and all. This lossless round-trip is the whole point: automated edits produce minimal, reviewable diffs instead of reformatting the file.

Recipes compose. A declarative recipe is a YAML list that names other recipes (`getRecipeList()`); an imperative recipe supplies a visitor directly. Larger migrations — Spring Boot 2→3, JUnit 4→5, a JDK version bump — are just recipes-of-recipes assembled from small ones. Catalog recipes live in separate artifacts (`rewrite-spring`, `rewrite-migrate-java`, `rewrite-testing-frameworks`, `rewrite-static-analysis`, …), each versioned independently, with `rewrite-recipe-bom` provided to align them.

**Rewrite 8 (2023) reworked the recipe-authoring API.** The older imperative style built on `doNext()` and mutable per-recipe visitor state was deprecated in favor of `getVisitor()`/`getRecipeList()`, and cross-file analysis moved to `ScanningRecipe` — a two-pass model where a recipe first accumulates information across all files, then edits[^4]. Recipes written against Rewrite 7 idioms generally need porting. This is the single largest source of "why doesn't this example compile" confusion for people learning from older blog posts.

Type attribution is what makes this expensive. To resolve symbols correctly the parser needs the project's compile classpath, so parsing is effectively coupled to your build's dependency resolution. The LST for a real project is large and is rebuilt in memory on every run in open-source OpenRewrite. Moderne's contribution is to build LSTs once and serialize them so they can be reused across runs, repos, and agents — the same serialized LST is what powers its agent tooling (type-aware search, recipes exposed as deterministic tool calls)[^3].

## Production Notes

**Memory and time scale with the codebase, not the change.** Even a one-line recipe pays the full parse-and-type-attribute cost. Large modules routinely need the JVM heap raised (`-Xmx`), and CI runs on big monorepos can OOM or take many minutes. Budget for this: OpenRewrite's runtime is dominated by parsing, not by the edit.

**Type attribution silently degrades if the classpath is incomplete.** When dependencies can't be resolved, the parser still produces an LST but with weaker or missing type information, and type-sensitive recipes then under-apply — they skip changes they should have made, without failing loudly. Symptoms of a broken migration are frequently a dependency-resolution problem upstream, not a recipe bug. Always confirm the project compiles cleanly first.

**Always `dryRun` before `run`.** Recipes rewrite source in place. The dry-run goals (`rewriteDryRun` / `rewrite:dryRun`) emit a patch you can review before applying. Treat recipe output like any large automated diff: apply on a branch, run the test suite, review.

**Recipe version alignment is a real chore.** Recipes ship as many independently versioned artifacts. Mismatched versions across `rewrite-spring`, `rewrite-migrate-java`, and the core cause obscure failures; use `rewrite-recipe-bom` and pin deliberately rather than sprinkling `latest.release`.

**Single-repo is the open-source boundary.** The Maven/Gradle plugins run one recipe against one repo. Fan-out across hundreds or thousands of repositories, serialized-LST reuse, and impact analysis are the commercial Moderne platform. Some catalog recipes are also Moderne-licensed even though the framework is Apache-2.0 — check licensing before assuming a given recipe is free to run at scale[^3].

**Non-Java languages are less mature.** Java is the anchor and by far the most complete. Kotlin, Groovy, JavaScript/TypeScript, Python, and C# parsers exist, but recipe coverage, type-attribution fidelity, and battle-testing trail Java, and running recipes for some of those languages depends on Moderne. Verify support depth for your specific language before committing a migration to it.

## When to Use / When Not

**Use when:**
- You have a repeatable, mechanical migration across a JVM codebase (framework/JDK upgrade, test-framework port, dependency bump) and want type-aware, minimal-diff edits.
- You want to encode a lint-and-fix rule once and apply it consistently, or author a custom recipe for an org-specific pattern.
- Correctness matters more than speed — you need edits that respect types and imports, not regex.

**Avoid when:**
- The edit is trivial and text-local (a string swap in a few files) — the LST parse cost isn't worth it; `sed`/`ast-grep`/IDE structural replace is faster.
- You need cross-repo, org-wide execution and aren't ready to adopt the commercial Moderne platform.
- Your codebase doesn't compile / can't resolve its classpath — type attribution will be poor and results unreliable.
- Your primary language is outside the well-supported set; coverage may not be there yet.

## Alternatives

- google/error-prone — Java compile-time bug detection with auto-fixes (Refaster templates); great for catching-and-patching antipatterns, weaker for large declarative migrations.
- ast-grep/ast-grep — polyglot structural search-and-rewrite in Rust; fast and no build integration, but pattern-based with no full type attribution.
- comby-tools/comby — language-agnostic structural find-and-replace; lighter than OpenRewrite, similarly not type-aware.
- facebook/jscodeshift — the standard for JavaScript/TypeScript codemods; use in JS-first shops instead of OpenRewrite's JS support.
- coccinelle (INRIA) — semantic patching for C; the closest analog outside the JVM world.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017–2019 | Originated at Netflix as an internal refactoring tool (Jonathan Schneider)[^1]. |
| public repo | 2020-05-12 | `openrewrite/rewrite` opened on GitHub; Moderne founded to steward it. |
| 7.x | 2021–2022 | LST + recipe/visitor model matured; Gradle & Maven plugins, growing catalog. |
| 8.0 | 2023 | Recipe API reworked: `getVisitor()`/`getRecipeList()`, `ScanningRecipe`; `doNext` deprecated[^4]. |
| 8.x | 2024–2026 | Multi-language parsers (Kotlin/Groovy/JS/Python/C#), broader catalog, agent-tool integration via Moderne. |

## References

[^1]: OpenRewrite README, "The OpenRewrite project (managed by Moderne)…" and project origin. https://github.com/openrewrite/rewrite
[^2]: OpenRewrite docs, "Lossless Semantic Trees." https://docs.openrewrite.org/concepts-and-explanations/lossless-semantic-trees
[^3]: OpenRewrite licensing / Moderne open-core boundary. https://docs.openrewrite.org/licensing/openrewrite-licensing
[^4]: OpenRewrite docs, "Migrating Rewrite 7 recipes to Rewrite 8." https://docs.openrewrite.org/changelog/migrating-to-rewrite-8

## Tags

java, refactoring, codemod, abstract-syntax-tree, static-analysis, jvm, code-migration, developer-tools, kotlin, apache-2.0
