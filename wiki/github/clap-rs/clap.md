# clap-rs/clap

> The de facto command-line argument parser for Rust — full-featured, and correspondingly heavy.

[GitHub repo](https://github.com/clap-rs/clap) ·
[Documentation](https://docs.rs/clap) ·
[License: MIT OR Apache-2.0](https://github.com/clap-rs/clap/blob/master/LICENSE-APACHE)

## Overview

clap (Command Line Argument Parser) is the most widely used argument-parsing
crate in the Rust ecosystem. It parses `std::env::args`, validates them against a
declared schema, generates `--help`/`--version` output, suggests corrections for
typos, and hands back typed values. Cargo, ripgrep, and a large share of Rust CLI
tooling depend on it directly or transitively. It was created by Kevin K.
(kbknapp) with a 1.0 release in 2015[^1].

clap offers two front-ends to the same engine: a runtime **builder API**
(`Command`/`Arg` objects assembled at startup) and a compile-time **derive API**
(`#[derive(Parser)]` on a struct). The derive layer was absorbed from the
separate `structopt` crate during the 3.0 rewrite[^2]; structopt is now
deprecated in favor of clap's built-in derive.

The defining tension is completeness versus weight. clap does subcommands,
argument groups, conflicts/requirements, value validation, environment-variable
fallback, shell completions, colored help, and Unicode-aware wrapping — very
little that a CLI needs is missing. That surface area costs compile time and
binary size. A hello-world clap binary is materially larger and slower to build
than one using a minimal parser, which is why a cottage industry of lighter
alternatives exists (see Alternatives). The 4.0 release (2022) was largely a
response to this: it split the crate into a workspace, gated more behind feature
flags, and trimmed default features to shrink the baseline[^3].

## Getting Started

```console
$ cargo add clap --features derive
```

```rust
// src/main.rs — derive API
use clap::Parser;

#[derive(Parser)]
#[command(version, about)]
struct Cli {
    /// Name to greet
    name: String,

    /// Times to repeat the greeting
    #[arg(short, long, default_value_t = 1)]
    count: u8,
}

fn main() {
    let cli = Cli::parse();
    for _ in 0..cli.count {
        println!("Hello, {}!", cli.name);
    }
}
```

The same parser in the builder API, for cases where the schema is dynamic:

```rust
use clap::{Arg, Command};

let matches = Command::new("greet")
    .arg(Arg::new("name").required(true))
    .arg(Arg::new("count").short('c').long("count").default_value("1"))
    .get_matches();
```

## Architecture / How It Works

clap 4 is a Cargo workspace, not a single crate[^3]:

- **`clap`** — the public facade; re-exports the builder and (behind the
  `derive` feature) the proc-macro.
- **`clap_builder`** — the runtime engine: the `Command`/`Arg` model, the
  matcher, validation, help/usage generation, and error rendering.
- **`clap_derive`** — the `#[derive(Parser)]`/`Args`/`Subcommand`/`ValueEnum`
  proc-macros, which generate builder calls at compile time. The derive API is
  therefore a thin front-end; both APIs converge on `clap_builder`.
- **`clap_lex`** — a minimal, dependency-light argv tokenizer. It is usable on
  its own for hand-rolled parsers.

Sibling crates live in the same org: **`clap_complete`** (bash/zsh/fish/
PowerShell/elvish completion scripts generated from a `Command`), **`clap_mangen`**
(roff man pages), and helpers such as `clap-verbosity-flag`.

Parsing is a two-phase process: the schema is built (once, at startup for the
builder API; generated code for derive), then argv is lexed and matched against
it, producing an `ArgMatches` map (builder API) or being deserialized into your
struct (derive API). Help and error text are formatted from the same `Command`
tree, which is why help output stays in sync with the actual accepted arguments.

Behavior is heavily feature-gated. Notable flags include `derive`, `cargo`
(pull crate metadata into `--version`/author), `env` (environment-variable
fallback), `unicode` (correct width for wide characters in help), `wrap_help`
(terminal-width-aware wrapping), `color`, `suggestions` (did-you-mean), and
`string` (owned dynamic strings in the builder). Several of these were on by
default in clap 3 and became opt-in in clap 4 to reduce the baseline.

## Production Notes

**Binary size and compile time are the real cost.** clap pulls in a nontrivial
amount of code, and with `derive` you also pay a proc-macro compile. For tools
where cold-build time or a small binary matters (embedded, WASM, tiny utilities,
CI-heavy repos), this is the reason to reach for a lighter parser. Mitigations
within clap: disable default features and re-enable only what you use, avoid
`wrap_help`/`unicode`/`color` if you don't need them, and prefer the builder API
when the proc-macro cost is the dominant factor.

**Breaking changes are concentrated at major versions and are large.** 2.x → 3.0
and 3.x → 4.0 both required real migration work: renamed types (`App` became
`Command`), removed constructors, changed default features, and dropped the YAML
config and the `clap_app!` macro. Pin the major version and read the migration
guide before bumping.

**Feature-flag surprises.** Because defaults were trimmed in 4.0, code that
"just worked" on clap 3 can lose colored help, `--version` author/description, or
env fallback simply because the feature is no longer default. If something in
help/version output disappears after an upgrade, check whether it now needs an
explicit feature.

**MSRV moves.** clap maintains a Minimum Supported Rust Version but advances it
over time; the `rust-version` field in `Cargo.toml` declares the current floor.
Projects that pin an old toolchain should verify the clap version they can use.

**Validation is your schema, not your logic.** clap validates arity, types,
conflicts, and requirements declaratively, but cross-argument business rules
(e.g. "if `--from` is set, `--to` is required *unless* `--all`") are cleaner
expressed with argument groups and `requires`/`conflicts_with` than with
post-parse `if` chains — though complex rules eventually spill into hand-written
checks regardless.

## When to Use / When Not

**Use when:**
- You want a full-featured CLI (subcommands, help, completions, env fallback)
  without building it yourself.
- Ergonomics and maintainability matter more than a few hundred KB of binary.
- You want the ecosystem-standard tool other Rust developers already know.

**Avoid when:**
- Binary size or compile time is a hard constraint (embedded, WASM, micro-CLIs).
- You need only a couple of flags — a minimal parser is faster to build and
  ships smaller.
- You want zero or near-zero dependencies.

## Alternatives

- clap-rs/clap with `--no-default-features` — same crate, trimmed; often enough.
- rust-cli/argh — Google's derive-based parser; smaller, opinionated (Fuchsia's
  style), fewer features. Use when binary size matters and you accept its syntax.
- pacak/bpaf — combinator + derive parser with strong composability; use when you
  want typed, composable parsers and are willing to learn the model.
- RazrFalcon/pico-args — tiny, zero-dependency, manual pull-based parsing. Use for
  the smallest possible footprint when you'll validate by hand.
- blyxxyz/lexopt — minimal argument *lexer*, not a full parser. Use when you want
  full control and near-zero overhead.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015 | Initial stable release[^1]. |
| 2.0 | 2016 | Long-lived stable line; builder API matured. |
| 3.0 | 2021-12 | Major rewrite; derive API absorbed from structopt[^2]. |
| 4.0 | 2022-09 | Workspace split, trimmed default features, smaller baseline; `App`→`Command`, YAML/`clap_app!` removed[^3]. |

## References

[^1]: clap changelog and release history. https://github.com/clap-rs/clap/blob/master/CHANGELOG.md
[^2]: structopt README — deprecation notice pointing to clap's derive API. https://github.com/TeXitoi/structopt
[^3]: clap 4.0 migration guide. https://github.com/clap-rs/clap/blob/master/docs/derive_ref/README.md and https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html

## Tags

rust, cli, argument-parser, command-line, derive-macro, subcommands, terminal, developer-tools
