# apple/pkl

> A configuration-as-code language from Apple: write config as a typed, validated program instead of hand-editing YAML.

[GitHub repo](https://github.com/apple/pkl) ·
[Official website](https://pkl-lang.org) ·
[License: Apache-2.0](https://github.com/apple/pkl/blob/main/LICENSE.txt)

## Overview

Pkl (pronounced "Pickle") is a domain-specific language for configuration, open-sourced by Apple in February 2024[^1]. It occupies the middle ground between static data formats (JSON, YAML) and general-purpose languages: config is written as Pkl source with classes, typed properties, constraints, functions, and template inheritance, then *rendered* to a target format — JSON, YAML, XML, `.plist`, `.properties`, or Pkl's own Pcf. The premise is that most YAML pain (no types, no validation, copy-paste across environments, stringly-typed everything) is better solved by a real language that produces data than by templating text.

Its defining tension is scope. Pkl is deliberately not Turing-complete-feeling general programming — it is oriented at config — but it is still a full language with a learning curve, an interpreter, and a runtime. Adopting it means every consumer of your config must either call Pkl to evaluate it or accept generated output checked into the repo. That is a real cost that flat YAML does not impose, and it is the axis on which Pkl succeeds or fails for a given team.

Pkl remains pre-1.0. Releases are on a `0.x` line, and the language and standard library still carry the possibility of breaking changes between minor versions[^2]. It is production-usable — Apple uses it internally — but it is not yet a frozen, stability-guaranteed spec.

## Getting Started

```bash
brew install pkl          # macOS/Linux via Homebrew
# or download a native CLI binary from the GitHub releases page
pkl --version
```

```pkl
// app.pkl — a typed, self-validating config module
class Server {
  hostname: String
  port: UInt16                    // range-checked at eval time
  poolSize: Int(this > 0) = 10    // constraint + default
}

server: Server = new {
  hostname = "0.0.0.0"
  port = 8080
}

// render this module to JSON:
//   pkl eval -f json app.pkl
```

```pkl
// prod.pkl — amend (inherit + override) the base module
amends "app.pkl"
server {
  hostname = "prod.internal"     // port, poolSize inherited unchanged
}
```

## Architecture / How It Works

Pkl's interpreter is built on GraalVM's Truffle language framework and runs on the JVM[^3]. Evaluation is a two-phase story: source parses and evaluates into an in-memory tree of typed values (with constraint checks and late-bound property resolution along the way), and a *renderer* then serializes that tree to the requested output format. Type checking and constraint validation happen during evaluation, so an invalid config fails before it ever produces output — the core value proposition.

The CLI ships two ways: a JVM jar and GraalVM `native-image` binaries. The native binaries start in milliseconds and are what most people install; the jar is the fallback where a native build for the platform/architecture is unavailable.

Language bindings (Go, Swift, Java, Kotlin) do not reimplement the evaluator. Instead the `pkl` binary runs as a subprocess "evaluator server" and the host language talks to it over a binary message-passing protocol[^4]. Your Go or Swift program hands Pkl a module path, Pkl evaluates and returns structured data, and codegen tools produce native structs/classes matching the Pkl schema. This keeps one canonical evaluator but means bindings inherit a process dependency on the CLI.

Modularity is first-class: modules `import` other modules, and `amends` performs template inheritance (an amended module is the parent with overrides applied). Dependencies can be pulled as *packages* over HTTPS, pinned by checksum. Standard-library modules (`pkl:json`, `pkl:yaml`, glob/regex/math helpers, etc.) are shipped with the runtime.

## Production Notes

**It is not sandboxed by default.** Pkl evaluation can read environment variables, read files, and make HTTP requests via resource/module readers. Evaluating an untrusted `.pkl` file is closer to running untrusted code than parsing untrusted JSON. Lock it down with `--allowed-modules` and `--allowed-resources` allowlists when evaluating anything you did not author. Treat this as a security boundary, not a convenience flag.

**Pre-1.0 churn.** Because the language is on a `0.x` line, standard-library APIs and occasional syntax can shift between minor releases. Pin your `pkl` version in CI and upgrade deliberately; do not float across `0.x` minors in a shared repo without reading release notes.

**Codegen drift.** When you use generated Go/Swift/Java classes, those artifacts must be regenerated whenever the Pkl schema changes. A stale generated struct that silently diverges from the `.pkl` source is a classic footgun — wire regeneration into the build or a CI check, don't do it by hand.

**JVM/startup and toolchain weight.** The native CLI mostly removes JVM cold-start pain, but the toolchain is still a GraalVM-derived binary of nontrivial size, and the jar path pays full JVM startup. In tight CI loops that evaluate config thousands of times, prefer the native binary and a single long-lived evaluator over per-call process spawns.

**Editor experience depends on plugins.** Meaningful authoring (type hints, go-to-definition, errors inline) requires the language server and an editor plugin — IntelliJ, VS Code, Neovim, and others are maintained as separate `apple/pkl-*` repos. Without them you are writing a typed language in a plain text editor and only learning about errors at `pkl eval` time.

**Ecosystem is young.** The package registry (`pkl-pantry` and friends) and third-party integrations are small compared to the YAML/Jsonnet/Helm world. For niche targets you may be writing the templates yourself.

## When to Use / When Not

**Use when:**
- Config is large, repeated across environments, or error-prone, and you want types + validation to fail fast at author time.
- You generate config for a typed consumer (Kubernetes manifests, app config structs in Go/Swift/Java/Kotlin) and want one schema with codegen.
- You want template inheritance (`amends`) instead of copy-pasting near-identical YAML per environment.

**Avoid when:**
- The config is small and static — plain JSON/YAML has zero runtime and zero learning curve, and Pkl buys you little.
- Your team won't adopt a new language, or downstream tooling can only consume raw YAML/JSON and you'd rather not check in generated output.
- You need a mature, 1.0-stable spec with a large third-party ecosystem today, or you must evaluate untrusted config without standing up a sandbox.

## Alternatives

- cue-lang/cue — types and values unified in one lattice; strong at validating existing YAML/JSON, less about template inheritance. Use instead when data validation and schema constraints matter more than programming ergonomics.
- google/jsonnet — mature, widely deployed templating superset of JSON. Use instead when you want ubiquity and templating without a type system.
- dhall-lang/dhall — total, non-Turing-complete functional config with strong guarantees. Use instead when you want maximal safety and are willing to pay a steeper learning curve.
- kcl-lang/kcl — a CNCF configuration language with validation, close in spirit to Pkl. Use instead when you want a comparable typed CaC language with heavier Kubernetes/cloud-native focus.
- hashicorp/hcl — config language of the Terraform ecosystem. Use instead when you live in Terraform/HashiCorp tooling and don't need Pkl's type system.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.25 | 2024-02 | First public open-source release under Apache-2.0[^1]. |
| 0.26–0.28 | 2024–2025 | Iterative `0.x` releases: stdlib additions, tooling, language-binding maturation[^2]. |
| — | 2026 | Still pre-1.0; actively maintained (~11.5k stars, frequent commits). |

## References

[^1]: Apple, "Pkl: Configuration that is Programmable, Scalable, and Safe" — open-source announcement, 2024-02-01. https://pkl-lang.org/blog/introducing-pkl.html
[^2]: Pkl release notes / changelog. https://pkl-lang.org/main/current/release-notes/index.html
[^3]: Pkl language reference and documentation (interpreter built on GraalVM Truffle; JVM and native CLI). https://pkl-lang.org/main/current/language-reference/index.html
[^4]: Pkl language bindings communicate with the evaluator over a message-passing protocol; see apple/pkl-go and apple/pkl-swift. https://github.com/apple/pkl-go

## Tags

java, jvm, graalvm, configuration-as-code, config-language, dsl, validation, kubernetes, apple, yaml-alternative, codegen
