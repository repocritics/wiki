# nickel-lang/nickel

> A gradually-typed configuration language with first-class contracts — "JSON with functions", built as an evolution of the Nix language.

[GitHub repo](https://github.com/nickel-lang/nickel) ·
[Official website](https://nickel-lang.org) ·
[License: MIT](https://github.com/nickel-lang/nickel/blob/master/LICENSE)

## Overview

Nickel is an interpreted configuration language whose job is to generate static
config — JSON, YAML, TOML, or arbitrary text — that some other system then
consumes[^1]. Its core is "JSON with functions": records, arrays, and scalars,
plus first-class functions, lazy evaluation, and a merge operator for composing
fragments. It was started at [Tweag](https://www.tweag.io) (now part of Modus
Create) and grew directly out of the Nix language, aiming to keep Nix's lazy
functional simplicity while adding typing, modularity, and independence from the
Nix package manager[^2]. The repo was originally `tweag/nickel` and now lives
under the community `nickel-lang` org; the reference interpreter is written in
Rust.

The defining bet is its type story: a *sound gradual type system* paired with
runtime **contracts**. Static types are opt-in and meant for the parts that
benefit most (reusable functions); configuration data itself is usually left
dynamic, validated by contracts that behave like schemas. Contracts are the
feature that distinguishes Nickel from "JSON with functions" languages like
Jsonnet — they are ordinary values you write, compose, and attach as
annotations, and they produce precise blame on failure rather than an opaque
type error deep in the evaluator.

The tension Nickel lives with is adoption versus theory. It is a well-designed,
actively-developed language (about 2.9k stars, still receiving commits in mid
2026) that nonetheless sits in a crowded niche next to CUE, Dhall, Jsonnet, KCL,
and Nix itself — none of which has decisively won. Choosing Nickel means betting
on a language whose ecosystem (editor tooling, generated schemas, importers) is
real but small, and whose most natural home — replacing the Nix language — is
also its hardest incumbent to displace.

## Getting Started

```bash
# with flake-enabled Nix
nix run nixpkgs#nickel -- repl
# macOS via Homebrew
brew install nickel
# from source (Rust toolchain)
cargo install nickel-lang-cli
```

```nickel
# config.ncl — a record with a contract, a default, and merge-friendly fields
let Port = std.contract.from_predicate (fun p => p >= 1 && p <= 65535) in
{
  host | String | default = "0.0.0.0",
  port | Port = 8080,
  url = "http://%{host}:%{std.to_string port}",
}
```

```console
$ nickel export --format json config.ncl
{
  "host": "0.0.0.0",
  "port": 8080,
  "url": "http://0.0.0.0:8080"
}
```

Annotations use the pipe syntax: `field | Contract | default = value`. A
contract violation (`port = 70000`) fails evaluation with blame pointing at the
offending field, not a stack trace.

## Architecture / How It Works

Nickel is a tree-walking interpreter, not a compiler. Source parses to an AST,
which is evaluated lazily (call-by-need) — fields are only forced when exported
or otherwise demanded. The pieces that matter:

- **Merge (`&`)** — the composition primitive, inspired by CUE's unification but
  adapted for a Turing-complete functional language. Two records merge field by
  field; conflicting concrete values are an error, but metadata (`default`,
  `optional`, contracts, documentation) has defined merge semantics. This is how
  Nickel does modularity and overriding without object-oriented inheritance.
- **Contracts** — runtime validators attached with `|`. A contract is a function
  from a value (plus a *label* carrying blame/position data) back to the value
  or an error. Because they are ordinary Nickel values, contracts compose,
  parameterize, and live in libraries. Record contracts double as schemas.
- **Gradual types** — the static checker runs only inside `: Type` annotated
  blocks; the boundary between typed and untyped code is guarded by an
  automatically-inserted contract, which is what makes the system *sound* rather
  than a best-effort linter[^1].
- **Stdlib** — shipped as `std.*` (string, array, record, contract, number
  modules), itself written largely in Nickel.
- **Tooling** — the workspace builds several crates: `nickel-lang-core`
  (evaluator), `nickel-lang-cli` (the `nickel` binary), and the Nickel Language
  Server (**NLS**) for editor diagnostics, type hints, and completion.
  Formatting is delegated to [Topiary](https://github.com/tweag/topiary).

The interpreter is deliberately kept simple and embeddable — it can be driven
from other languages via the core crate. The trade-off is performance: a naive
lazy tree-walker re-evaluates aggressively, which is why the roadmap centers on
a bytecode VM and incremental evaluation (see below).

## Production Notes

- **Performance is the known weak spot.** For small-to-medium configs it is
  fine, but large merged configurations can be slow, and there is no caching
  between runs. The team's stated next steps are a bytecode compiler + VM
  (RFC007) and incremental re-evaluation[^3] — meaning current performance is a
  known limitation being actively worked, not a solved problem.
- **Contracts run at evaluation time.** Validation cost is paid every eval;
  heavy contract use on large data shows up in runtime. There is no separate
  compile-then-validate phase.
- **Small ecosystem, real but thin.** First-party projects exist for Kubernetes
  contracts (`nickel-kubernetes`), Terraform (`tf-ncl`), Bazel (`rules_nickel`),
  JSON-Schema import (`json-schema-to-nickel`), and dev environments (Organist).
  Outside these, you will often be the first to write a contract for your target
  schema.
- **Nix is the flagship use case but not a drop-in replacement.** Nickel is
  positioned as an evolution of the Nix *language*, yet nixpkgs is written in Nix
  and integration remains a project-level effort, not a switch you flip.
- **Language still evolving post-1.0.** The core has been stable since 1.0 (May
  2023), but features like custom merge functions are still in RFC/proposal
  stage[^3]. Pin your interpreter version in CI; stdlib and formatter output can
  shift across releases.
- **Error messages are a genuine strength.** Blame-tracking contracts and typed
  boundaries produce diagnostics that point at the source of a bad value, which
  is a real operational advantage over YAML-plus-templating stacks.

## When to Use / When Not

**Use when:**
- You generate complex config (infra, Kubernetes, build systems) and want
  functions, schemas, and validation instead of string-templated YAML.
- You want schema validation as first-class values you can compose and reuse.
- You like the Nix mental model (lazy, functional, merge-based) but want typing
  and independence from the package manager.

**Avoid when:**
- Your config is small and static — plain JSON/YAML or a lighter templater is
  less to learn and deploy.
- You need maximum evaluation speed or caching today — the VM/incremental work
  is roadmap, not shipped.
- You need a large mature ecosystem of pre-written schemas and integrations —
  CUE and Jsonnet have more real-world adoption.
- You want strict, fully-checked static typing everywhere — Dhall enforces that;
  Nickel intentionally does not.

## Alternatives

- cue-lang/cue — constraint/unification-based validation, no general functions or
  Turing-completeness; use when data validation and schema rigor matter more than
  programmability.
- dhall-lang/dhall — statically typed, total (non-Turing-complete) config; use
  when you want the type system to check everything and accept mandatory
  annotations.
- google/jsonnet — the established "JSON with functions" with object-style
  inheritance and no typing; use when you want a proven, widely-adopted templater.
- kcl-lang/kcl — gradually typed, schema/OO-oriented, strict evaluation; use when
  you want gradual typing but prefer inheritance-based schemas and eager eval.
- NixOS/nix — the incumbent Nickel evolves from; use when you are already inside
  the Nix package/derivation world and don't need to leave it.

## History

| Version | Date | Notes |
|---------|------|-------|
| Public announcement | 2020-03 | Tweag introduces Nickel; interpreter prototype in Rust[^2]. |
| 1.0 | 2023-05 | First stable release; core language design frozen as stable[^3]. |
| Org move | ~2024 | Repo moves from `tweag/nickel` to community `nickel-lang` org. |
| Active dev | 2026-07 | Still receiving commits; work on bytecode VM (RFC007) and incremental evaluation ongoing[^3]. |

## References

[^1]: Nickel README and language overview — gradual typing, contracts, merge. https://github.com/nickel-lang/nickel
[^2]: Tweag, "Nickel: better configuration for less" — project origin and Nix relationship. https://www.tweag.io/blog/2020-03-04-nickel/
[^3]: Nickel README, "Current state and roadmap" — 1.0 (May 2023), bytecode interpreter RFC007, incremental evaluation, custom merge functions. https://github.com/nickel-lang/nickel/blob/master/rfcs/007-bytecode-interpreter.md

## Tags

configuration-language, config, rust, gradual-typing, contracts, nix, infrastructure-as-code, functional, dsl, json
