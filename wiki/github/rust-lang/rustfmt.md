# rust-lang/rustfmt

> The official Rust code formatter — one opinionated style, distributed with the toolchain.

[GitHub repo](https://github.com/rust-lang/rustfmt) ·
[Official website](https://rust-lang.github.io/rustfmt/) ·
[License: MIT OR Apache-2.0](https://github.com/rust-lang/rustfmt/blob/main/LICENSE-APACHE)

## Overview

rustfmt is the canonical formatter for Rust source, maintained inside the
rust-lang organization and shipped as a `rustup` component alongside every
toolchain[^1]. It reformats code to conform to the Rust Style Guide, a
specification that was formalized through a dedicated style RFC process rather
than invented ad hoc by the tool's authors[^2]. Because the style is governed
separately from the implementation, rustfmt is closer to `gofmt` in philosophy
than to Prettier: the goal is one community-wide default, not a per-project
aesthetic. Most Rust projects run `cargo fmt` in CI and treat the output as
non-negotiable.

The defining tension is **configurability versus stability**. rustfmt exposes a
large `rustfmt.toml` surface, but the majority of the interesting options
(`imports_granularity`, `group_imports`, `wrap_comments`,
`format_code_in_doc_comments`, and many others) are marked *unstable* and only
take effect on a nightly toolchain[^3]. On stable Rust you are largely limited
to the defaults. This is deliberate — the project does not want to lock in
formatting decisions it may later revise — but in practice it is the single
most common source of user frustration: teams adopt an option, discover it
silently does nothing on stable, and end up pinning nightly just to format.

A second structural fact shapes everything: rustfmt parses code with the actual
Rust compiler's parser, so it is version-coupled to the toolchain and cannot
format code that does not parse. Formatting is a whole-program guarantee, not a
line-by-line one.

## Getting Started

Install as a component and run through Cargo:

```sh
rustup component add rustfmt
cargo fmt                 # format the whole crate/workspace in place
cargo fmt --all -- --check   # CI mode: exit 1 if anything is unformatted
```

For the unstable options, use nightly:

```sh
rustup component add rustfmt --toolchain nightly
cargo +nightly fmt
```

A `rustfmt.toml` (or `.rustfmt.toml`) in the project root or any parent
directory configures behavior. Generate the defaults and edit down:

```sh
rustfmt --print-config default rustfmt.toml
```

```toml
# rustfmt.toml
edition = "2021"          # parsing edition (see footgun below)
style_edition = "2024"    # formatting rules revision
max_width = 100
```

Opt individual items out with `#[rustfmt::skip]`.

## Architecture / How It Works

rustfmt is not a standalone parser. It consumes `rustc`'s own AST (historically
via the published `rustc-ap-*` crates, later as an in-tree component), which is
why it stays in lockstep with the compiler and why a single syntax error aborts
formatting for the entire file[^4]. The pipeline is: parse source to AST, walk
the AST emitting a normalized token stream through a width-aware line-wrapping
engine (`max_width`, `comment_width`, indentation rules), then splice the
result back around regions the tool refuses to touch.

Several node kinds are treated as opaque. rustfmt does **not** reformat comment
bodies, most macro *definitions*, and code inside doc-comments unless the
corresponding unstable options are enabled. Macro *invocations* are formatted
only in a subset of cases. Non-ASCII-heavy code is handled but explicitly
outside the stability guarantee per the project's own limitations list[^5].

Two edition concepts sit side by side and are frequently conflated:

- **`edition`** selects the language grammar used for parsing. It changes what
  parses, not how output looks.
- **`style_edition`** selects the *formatting* rules revision (2015/2018/2021/
  2024), introduced by RFC 3338's "style evolution" mechanism so the default
  style can change over Rust releases without reformatting the entire
  ecosystem overnight[^6]. The 2024 style edition ships with Edition 2024.

The repository is the development home for rustfmt but is kept in sync with the
`rust-lang/rust` tree, which is what allows the component to be released on the
same cadence as the compiler.

## Production Notes

**Stable vs nightly is the first decision, not a detail.** Any `rustfmt.toml`
that relies on an unstable option will be a no-op on stable and will only be
honored under `cargo +nightly fmt`. Check `rustfmt --help=config` and confirm an
option's stability before standardizing on it across a team.

**The `rustfmt` vs `cargo fmt` edition mismatch is a real footgun.** Invoked
directly, `rustfmt` defaults to `edition = 2015`; `cargo fmt` infers the edition
from `Cargo.toml`. The same for `style_edition`. Editor integrations and
pre-commit hooks that shell out to the `rustfmt` binary can therefore produce
different output than CI's `cargo fmt`, causing check failures that look like
formatting drift. The fix is to pin both `edition` and `style_edition`
explicitly in `rustfmt.toml`[^7].

**A single parse error stops the file.** Because formatting runs on the
compiler's parser, work-in-progress code with a missing brace won't be
formatted at all — not partially. This surprises people expecting Prettier-style
best-effort formatting.

**Comments and macros are largely untouched.** Long comment reflow requires the
unstable `wrap_comments`; macro-heavy code (e.g. `html!`-style DSLs) will not be
tidied and often needs `#[rustfmt::skip::macros(...)]`.

**CI integration** is via `--check` (exit 1 on differences). This is stable and
the standard gate; the older `--write-mode diff` is obsolete. Machine-readable
output (`--emit json`/`checkstyle`) exists but several emit modes are
nightly-only.

**Version pinning matters for reproducibility.** Formatting output can change
between toolchain releases within a style edition (bug fixes are explicitly not
covered by the stability guarantee). Teams that need byte-identical CI results
across machines should pin the toolchain via `rust-toolchain.toml`.

## When to Use / When Not

**Use when:**
- You want the community-standard Rust style with zero bikeshedding — the
  default path (`rustup component add rustfmt` + `cargo fmt`) is the answer for
  essentially every Rust project.
- You need an enforceable CI gate (`cargo fmt --all -- --check`).
- You want editor-on-save formatting via rust-analyzer, which drives rustfmt.

**Avoid / reconsider when:**
- You expect deep per-project style control on **stable** — most knobs are
  nightly-only; if that is unacceptable, either commit to nightly rustfmt or
  accept the defaults.
- You need comment/doc-comment reflow or aggressive macro formatting on stable —
  those live behind unstable options.
- You want best-effort formatting of code that doesn't yet compile — rustfmt
  requires a clean parse.

## Alternatives

- rust-lang/rust-clippy — complementary, not an alternative: it lints for
  correctness and idioms; run it alongside rustfmt, not instead of it.
- dprint/dprint — reach for it in polyglot monorepos needing one formatter
  across TOML/Markdown/JS as well as Rust; note its Rust support wraps rustfmt
  rather than replacing it.
- astral-sh/ruff — the Python-ecosystem analog (opinionated formatter plus
  linter); use it when the codebase is Python, not Rust.
- psf/black — Python's "uncompromising" formatter, the closest cross-language
  design cousin to rustfmt's one-true-style philosophy.
- golang/go (`gofmt`) — the design ancestor of the whole zero-config-formatter
  approach; the reference point if you want to understand rustfmt's stance.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015 | Started by Nick Cameron as an external tool early in Rust 1.x[^1]. |
| 0.x | 2016–2018 | Iteration on style; usable but pre-stability. |
| 1.0 | 2018-12 | First stable release, aligned with the Rust 2018 edition; formatting stability guarantee for whole programs begins[^8]. |
| style RFC 3338 | 2022 | "Style evolution" accepted, introducing style editions so defaults can change over time[^6]. |
| style_edition 2024 | 2025 | 2024 style edition ships with Edition 2024 (Rust 1.85 era). |

## References

[^1]: rustfmt README, "Quick start" — install/run via `rustup component add rustfmt` and `cargo fmt`. https://github.com/rust-lang/rustfmt#quick-start
[^2]: Rust Style Guide. https://doc.rust-lang.org/nightly/style-guide/
[^3]: rustfmt Configurations reference (stable vs unstable options; unstable require nightly). https://rust-lang.github.io/rustfmt/
[^4]: rustfmt README, "Running" and build notes — rustfmt operates on parsed Rust and is coupled to the compiler front end. https://github.com/rust-lang/rustfmt#running
[^5]: rustfmt README, "Limitations" — non-parsing code, comments, macros, and non-ASCII outside the stability guarantee. https://github.com/rust-lang/rustfmt#limitations
[^6]: RFC 3338, "Rust style evolution". https://rust-lang.github.io/rfcs/3338-style-evolution.html
[^7]: rustfmt README, "Rust's Editions" / "Style Editions" — `rustfmt` defaults to edition 2015 while `cargo fmt` infers from `Cargo.toml`. https://github.com/rust-lang/rustfmt#configuring-rustfmt
[^8]: fmt-rfcs, the style RFC process repository. https://github.com/rust-dev-tools/fmt-rfcs

## Tags

rust, formatter, code-formatter, cli, developer-tools, style-guide, cargo, rustup, static-analysis, toolchain
