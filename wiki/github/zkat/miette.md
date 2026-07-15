# zkat/miette

> A diagnostic library for Rust that extends `std::error::Error` with error codes, source-span labels, and a compiler-grade graphical report renderer.

[GitHub repo](https://github.com/zkat/miette) ·
[Documentation](https://docs.rs/miette) ·
[License: Apache-2.0](https://github.com/zkat/miette/blob/main/LICENSE)

## Overview

miette (pronounced like the name) is a Rust error-reporting library written by Kat Marchán (zkat), who grew it out of her Rust package-manager tooling — the README still references `ruget`, an early npm-for-.NET experiment[^1]. Its central idea is a `Diagnostic` trait that is a strict superset of `std::error::Error`: any diagnostic is still a plain error, so a library can return miette types without forcing that choice on its consumers[^2]. On top of the error, `Diagnostic` layers optional metadata — a stable error code, a help string, a documentation URL, a severity, and, most distinctively, labeled spans into an associated `SourceCode`. The payoff is `rustc`-style output: the offending snippet printed with underlines, arrows, and cause chains.

The defining tension is the `fancy` feature flag. The graphical renderer pulls in a stack of terminal-detection and text-layout crates (color support, hyperlink support, unicode width, wrapping), so miette splits itself in two: the trait/derive layer that libraries depend on, and the `fancy` rendering layer that only the top-level binary should enable. Getting this split wrong — a library turning on `fancy` — is the single most common miette mistake, and the README warns about it twice[^3].

miette occupies a middle ground between `anyhow`/`eyre` (dynamic, app-level error wrappers with no notion of source spans) and full compiler diagnostic crates like `codespan-reporting` or `ariadne` (span rendering with no error-trait integration). It is the default diagnostic layer for a good deal of Rust parser and compiler-adjacent tooling.

## Getting Started

```sh
cargo add miette --features fancy   # in your binary crate
cargo add thiserror                  # the usual companion for deriving errors
```

```rust
use miette::{Diagnostic, NamedSource, SourceSpan, Result};
use thiserror::Error;

#[derive(Error, Debug, Diagnostic)]
#[error("oops!")]
#[diagnostic(code(oops::my::bad), help("try doing it better next time?"))]
struct MyBad {
    #[source_code]
    src: NamedSource<String>,
    #[label("this bit here")]
    bad_bit: SourceSpan,
}

fn main() -> Result<()> {
    let src = "source\n  text\n    here".to_string();
    Err(MyBad {
        src: NamedSource::new("bad_file.rs", src),
        bad_bit: (9, 4).into(), // byte offset 9, length 4
    })?;
    Ok(())
}
```

Returning `miette::Result` from `main` prints the graphical report automatically — because that path renders the `Report` via its `Debug` (`{:?}`) impl, not `Display`.

## Architecture / How It Works

The crate is three layers stacked on `std::error::Error`:

1. **The `Diagnostic` trait** — methods (`code`, `help`, `url`, `severity`, `labels`, `source_code`, `related`, `diagnostic_source`) that all default to `None`, so implementing it is no heavier than implementing `Error`. Each maps to one region of the rendered report.
2. **The derive macro** (`miette-derive` crate) — turns `#[diagnostic(...)]` and `#[label]`/`#[source_code]`/`#[related]` field attributes into that trait impl. It is designed to compose with `thiserror`'s `#[error(...)]` derive; miette does not define the `Display`/`Error` half itself, so the two macros are almost always used together.
3. **`Report` and the handler layer** — `Report` is a type-erased, heap-allocated boxed diagnostic (the `anyhow::Error` analogue), and `IntoDiagnostic`/`WrapErr`/`Context` convert foreign errors into it. Rendering is pluggable through the `ReportHandler` trait; `set_hook` installs a global handler once per process. `fancy` ships two: `GraphicalReportHandler` (ANSI/Unicode) and `NarratableReportHandler` (screen-reader/braille prose).

Spans are the interesting internal detail. A `SourceSpan` is a byte offset plus length into a `SourceCode`; `SourceCode::read_span` returns `SpanContents`, which the handler uses to slice out context lines and place carets. `SourceCode` is implemented for `String`/`&str` out of the box, and `NamedSource` attaches a filename. Because offsets are byte-based, the renderer — not the caller — handles UTF-8 boundaries and line splitting.

The narratable handler is not just an accessibility afterthought: miette auto-switches to it under `NO_COLOR`, `CLICOLOR`, and CI heuristics, so the same diagnostic renders as graphical art locally and as linear prose in a CI log.

## Production Notes

- **`Debug`, not `Display`, is the rendered output.** `println!("{}", report)` prints one terse line; `println!("{:?}", report)` prints the full graphical report. This inversion of the usual Rust convention trips up nearly everyone once.
- **Never enable `fancy` in a library.** It is additive across the dependency graph, so one library turning it on forces its terminal-detection and layout dependencies onto every downstream consumer. Libraries depend on plain `miette`; only the final binary adds `features = ["fancy"]`.
- **Byte offsets are a footgun.** Labels are byte spans. Computing them from character indices (e.g. `str::chars().count()`) mis-highlights or panics on multi-byte UTF-8. Use byte offsets from the same source the label points into.
- **`set_hook` is one-shot.** It returns an error if a handler is already installed, and it races with anything else in the process trying to set it (test harnesses, `color-eyre`-style panic hooks). Set it once, early, in `main`.
- **`Report` requires `Send + Sync + 'static`.** Wrapping a non-`Sync` error into a `Report` will not compile — a common surprise when converting from `Box<dyn Error>`.
- **CI output changes shape.** Because the narratable printer activates on CI/`NO_COLOR`, the exact rendered text differs between local dev and CI, which breaks naive snapshot tests of error output.
- **Upgrade churn across majors.** The trait and macro surface has been re-cut several times; the 7.0 line made `NamedSource` generic (`NamedSource<T>`) and trimmed/refreshed the `fancy` dependency set, both of which are source-breaking for code that named the old concrete types. Pin the major and read the changelog before bumping.
- **std only.** miette builds on `std::error::Error`, so there is no `no_std` mode.

## When to Use / When Not

**Use when:**
- You are writing a parser, compiler, linter, or config loader and want to point at exact spans in user-supplied source.
- You want `anyhow`-style ergonomics (`Report`, `into_diagnostic()`, `wrap_err`) but with error codes and labeled snippets.
- You need accessible/CI-friendly error output for free alongside the pretty version.

**Avoid when:**
- You just need dynamic error propagation with backtraces and no spans — `anyhow` or `eyre` are lighter.
- You are a library and would have to expose miette in your public API for callers who don't want it (return concrete `thiserror` enums instead; adopting `Diagnostic` is cheap to add later).
- You target `no_std`, or you want the diagnostic renderer without the `std::error::Error` coupling — use a rendering-only crate.

## Alternatives

- dtolnay/anyhow — dynamic app-level error type; use instead when you want propagation and backtraces but have no source spans to point at.
- dtolnay/thiserror — derive for concrete error enums; not a competitor but the standard companion, and sufficient alone when you don't need rendering.
- yaahc/eyre — customizable `anyhow` fork (`color-eyre` reporter); use when you want pluggable reporting hooks without miette's span/label model.
- zesterer/ariadne — span-based diagnostic renderer with rich output; use when you want the report art but not an error trait.
- brendanzab/codespan — `codespan-reporting` for language tooling; use for compiler-style diagnostics decoupled from `std::error::Error`.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-08 | Repo created; grew out of zkat's Rust package-manager tooling[^1]. |
| 1.x | 2021 | `Diagnostic` trait, derive macro, `fancy` graphical + narratable handlers. |
| 3.x | 2022 | `Report`/`IntoDiagnostic`/`WrapErr`, `miette!`/`bail!`/`ensure!` macros. |
| 5.x | 2022 | Handler/`SourceCode` refinements; widely adopted baseline. |
| 7.x | 2024 | `NamedSource<T>` generic, `fancy` dependency refresh, MSRV bump[^4]. |

## References

[^1]: zkat/miette README — "About" and `ruget` reference. https://github.com/zkat/miette
[^2]: `miette::Diagnostic` trait docs. https://docs.rs/miette/latest/miette/trait.Diagnostic.html
[^3]: zkat/miette README — `fancy` feature warning ("You should only do this in your toplevel crate"). https://github.com/zkat/miette
[^4]: miette crate release history on crates.io. https://crates.io/crates/miette/versions

## Tags

rust, error-handling, error-reporting, diagnostics, cli, source-spans, derive-macro, thiserror, compiler-tooling, terminal
