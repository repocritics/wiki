# remkop/picocli

> A single-file Java framework for command-line apps — annotation-driven parsing, subcommands, ANSI help, and GraalVM native-image support.

[GitHub repo](https://github.com/remkop/picocli) ·
[Official website](https://picocli.info) ·
[License: Apache-2.0](https://github.com/remkop/picocli/blob/main/LICENSE)

## Overview

Picocli is a command-line argument parsing framework for the JVM, written by Remko Popma and first released as 1.0 in 2017[^1]. You annotate a class with `@Command`, `@Option`, and `@Parameters`; picocli populates fields from `args[]`, converts strings to strongly typed values, generates a usage-help screen, and dispatches to your `Runnable`/`Callable`. It targets Java but is routinely used from Groovy, Kotlin, and Scala.

Two design decisions define it. First, the entire library ships in a **single source file** (`CommandLine.java`, tens of thousands of lines): you can drop it into your project as source and have zero external dependencies at runtime, which matters for tools that must not drag a dependency tree behind them. Second, it is built to be **GraalVM native-image friendly** — picocli ships an annotation processor that generates the reflection/resource config Graal needs, so a picocli CLI can be ahead-of-time compiled to a self-contained native executable with millisecond startup[^2]. That combination — no runtime deps plus native-image support — is why it displaced Apache Commons CLI and JCommander as the default choice for new JVM CLIs.

The tradeoff is surface area. Picocli is feature-maximalist: negatable options, argument groups (mutually exclusive/dependent), `@-files`, map options, arity ranges, custom type converters, `@Spec` model access, repeating composite groups, and more. The user manual is book-length. For a two-flag script this is overkill; the payoff appears once a CLI grows subcommands, validation, and polished help.

## Getting Started

Maven coordinates: `info.picocli:picocli` (latest 4.7.7). Gradle:

```groovy
dependencies { implementation 'info.picocli:picocli:4.7.7' }
```

```java
import picocli.CommandLine;
import picocli.CommandLine.Command;
import picocli.CommandLine.Option;
import picocli.CommandLine.Parameters;
import java.util.concurrent.Callable;

@Command(name = "greet", mixinStandardHelpOptions = true, version = "greet 1.0")
public class Greet implements Callable<Integer> {

    @Parameters(index = "0", description = "Who to greet.")
    private String name;

    @Option(names = {"-c", "--count"}, description = "Repeat count.")
    private int count = 1;

    @Override
    public Integer call() {
        for (int i = 0; i < count; i++) System.out.println("Hello, " + name);
        return 0;
    }

    public static void main(String[] args) {
        System.exit(new CommandLine(new Greet()).execute(args));
    }
}
```

`mixinStandardHelpOptions` adds `-h/--help` and `-V/--version` for free. `execute()` handles parsing, help/version short-circuits, error reporting, and returns the exit code.

## Architecture / How It Works

Parsing is a two-phase process over a **command object model**. Annotations on your class are reflected into a tree of `CommandSpec` / `OptionSpec` / `PositionalParamSpec` objects at construction time; the parser then walks `args[]` against that model, applying type converters and arity rules, and assigns results back to your fields (or calls setters). The same model is reachable programmatically via a builder API, so the annotations are sugar over a model you can also construct by hand — useful for dynamic commands where options aren't known at compile time.

Key internals:

- **Type conversion** — a registry of `ITypeConverter` handles built-ins (numbers, enums, `File`, `Path`, network types); you register converters for your own types. Collections/arrays/maps are populated according to declared arity (`0..*`, `1..2`, etc.).
- **Subcommands** — commands nest into a tree (`git`-style). Each subcommand is its own annotated class or `CommandSpec`; the parser routes remaining args down the tree and can run every command in the chain.
- **Argument groups** — `@ArgGroup` encodes mutual exclusion, co-dependence, and help-section grouping declaratively, which is where much of the parser's validation complexity lives.
- **Usage help** — a `Help` layer renders columns, wraps text to terminal width, and emits ANSI styles via a `Help.Ansi` abstraction that auto-detects TTY/`NO_COLOR`/Windows support.
- **GraalVM support** — because native-image forbids runtime reflection unless declared, picocli's annotation processor (`picocli-codegen`) emits `reflect-config.json` and friends at compile time so the model survives AOT compilation.

Optional companion modules live in the same repo: `picocli-spring-boot-starter`, `picocli-shell-jline2`/`jline3` for interactive shells, `picocli-codegen`, and Groovy support. The core, however, is deliberately dependency-free.

## Production Notes

**Native-image config is the top footgun.** The annotation processor generates Graal config only for what it can see at compile time. Dynamically registered subcommands, reflectively loaded converters, or resource bundles often need hand-written `reflect-config.json`/`resource-config.json` entries. Symptom: the CLI works on the JVM and then throws `ClassNotFoundException`/missing-resource at native runtime. Verify against a real `native-image` build in CI, not just `java -jar`.

**Reflection at scale.** On the plain JVM, building the model reflects over your annotated classes on every startup. For a native image this cost is gone; on the JVM it's usually negligible but can show up if you construct many commands per invocation.

**Java baseline.** Picocli deliberately keeps a very low minimum Java version (historically Java 5/6-era bytecode) so it runs everywhere, and it degrades gracefully rather than requiring Java 8+. That conservatism is a feature for library authors but means the API predates modern conveniences.

**Bus factor.** Development is overwhelmingly one person (Remko Popma). The project is mature, well-tested (high coverage, CI across many Java versions), and responsive, but single-maintainer risk is real for anything you're betting a product on — budget for the possibility of slower releases.

**Exit codes and testing.** Prefer `execute()` (returns an exit code) over the older `parseArgs`/`run` helpers; hand-rolling `System.exit` inside `call()` makes commands hard to unit-test. Inject `ParseResult` or use `@Spec` to keep logic testable.

**Framework integration.** Micronaut and Quarkus both have first-class picocli command modes, and there's a Spring Boot starter — but each wires dependency injection differently, and DI-managed command instances interact with picocli's own instantiation. Read the integration guide for your framework rather than assuming the vanilla `new CommandLine(obj)` path.

## When to Use / When Not

**Use when:**
- You're building a JVM CLI with subcommands, typed options, and polished help.
- You want a GraalVM native executable with fast startup and no JVM warmup.
- You need zero runtime dependencies (single-file source include).
- You're on Kotlin/Groovy/Scala and want annotation-driven parsing.

**Avoid when:**
- The tool takes one or two flags — `args[0]` or a tiny hand-parser is simpler.
- You're not on the JVM (this is Java-only; see clap/cobra/argparse for other stacks).
- You need a GUI or interactive TUI as the primary interface (picocli is argument parsing; pair with JLine for shells).

## Alternatives

- cbeust/jcommander — the older annotation-based Java parser; lighter, fewer features, no native-image tooling. Use when you want minimal surface area on the JVM.
- apache/commons-cli — the venerable imperative parser. Use when you already depend on it or want maximal stability over features; expect boilerplate.
- ajalt/clikt — Kotlin-first CLI framework. Use when your project is Kotlin and you prefer its idioms over annotations.
- spf13/cobra — the Go standard for subcommand CLIs (kubectl, gh). Use when you're building in Go, not the JVM.
- clap-rs/clap — Rust's derive-based parser with native binaries by default. Use when you want single-binary CLIs without a JVM at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017-08 | First stable release; annotations API[^1]. |
| 2.0 | 2018-01 | Subcommand improvements, Groovy script support. |
| 3.0 | 2018-07 | Programmatic API / command model, mixins. |
| 4.0 | 2019-06 | Execution framework (`execute`), exit codes, `@ArgGroup`. |
| 4.7.0 | 2022-11 | Negatable options refinements, i18n/help improvements. |
| 4.7.7 | 2025 | Current maintenance release of the 4.7 line[^3]. |

## References

[^1]: "Announcing picocli 1.0" — picocli.info. https://picocli.info/announcing-picocli-1.0.html
[^2]: "Picocli on GraalVM: Blazingly Fast Command Line Apps" / GraalVM native-image guide. https://picocli.info/picocli-on-graalvm.html
[^3]: picocli releases — latest tag v4.7.7. https://github.com/remkop/picocli/releases

## Tags

java, cli, argument-parsing, command-line, graalvm, native-image, jvm, annotations, subcommands, kotlin, framework
