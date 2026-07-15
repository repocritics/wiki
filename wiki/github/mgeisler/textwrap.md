# mgeisler/textwrap

> A Rust library for wrapping, filling, and indenting text — with optional optimal-fit line breaking and TeX-quality hyphenation.

[GitHub repo](https://github.com/mgeisler/textwrap) ·
[crates.io](https://crates.io/crates/textwrap) ·
[docs.rs](https://docs.rs/textwrap/) ·
[License: MIT](https://github.com/mgeisler/textwrap/blob/main/LICENSE)

## Overview

Textwrap is a small, single-purpose Rust crate that turns a long string into a
sequence of lines that fit a given width. Its most common consumer is
command-line tooling: formatting `--help` output, wrapping error messages, and
laying out tabular or paragraph text so it reads well in a terminal. It also
works for proportional fonts, which is why it ships a WebAssembly demo that
wraps sans-serif, serif, and monospace text on an HTML canvas.

The library's defining choice is that it does not wrap one line at a time. With
the `smawk` feature enabled (the default), it uses an *optimal-fit* algorithm
that looks ahead across the whole paragraph and picks break points that
minimize the raggedness left at line ends — the same Knuth-Plass idea TeX uses
for typesetting[^1]. This produces visibly more balanced output than the naive
greedy approach, at the cost of a dependency and slightly more work per wrap.

The central tension is scope versus weight. Textwrap wants to be the crate you
reach for without thinking, but "wrap text correctly for every human language"
is a surprisingly large problem: display width of CJK and emoji, line-break
opportunities from Unicode UAX #14, and hyphenation dictionaries all pull in
data and dependencies. Textwrap resolves this by making almost everything an
opt-in Cargo feature, so a minimal build is genuinely small while a
full-featured one is correct across scripts[^2].

## Getting Started

```toml
[dependencies]
textwrap = "0.16"
```

```rust
use textwrap::{fill, wrap};

let text = "textwrap: an efficient and powerful library for wrapping text.";

// wrap returns the lines as a Vec<Cow<str>>
assert_eq!(
    wrap(text, 28),
    vec![
        "textwrap: an efficient",
        "and powerful library for",
        "wrapping text.",
    ],
);

// fill joins them back with newlines
println!("{}", fill(text, 28));
```

The first line above is only 22 columns wide even though "and" would fit — that
is the optimal-fit algorithm choosing a more even distribution over a longer
first line. Disable the `smawk` feature (or set
`Options::wrap_algorithm(WrapAlgorithm::FirstFit)`) to get the greedy behavior,
where each line is packed as full as possible.

## Architecture / How It Works

The public API is a thin layer over a configurable `Options` struct and a
lower-level `core` module. `wrap`, `fill`, `indent`, `dedent`, and `refill` are
the convenience entry points; everything they do is expressible through
`Options`, which controls:

- **`width`** — the target column count. `termwidth()` (behind the
  `terminal_size` feature) reads it from the current terminal.
- **`wrap_algorithm`** — `OptimalFit` (via the separate `smawk` crate, also by
  the author) or `FirstFit`. OptimalFit runs the SMAWK algorithm to find
  minimum-cost breaks in linear time relative to the number of words[^1].
- **`word_separator`** — how the input is split into breakable units. With
  `unicode-linebreak` it follows UAX #14 line-break opportunities; without it,
  splitting is on ASCII spaces only.
- **`word_splitter`** — what to do with a single word longer than `width`.
  Options are hard-breaking, no splitting, or `Hyphenation`, which consults a
  loaded `hyphenation` dictionary.
- **`initial_indent` / `subsequent_indent`** — per-line prefixes, which is how
  bullet lists and hanging indents are built.

Display width is computed with `unicode-width` so that double-width CJK
characters and zero-width combining marks count correctly rather than as one
column each. This matters: naive `.len()` or `.chars().count()` wrapping is
wrong for any non-Latin text, and textwrap's correctness here is a large part
of why people depend on it.

Hyphenation is deliberately kept at arm's length. The `hyphenation` feature
pulls in the external crate of the same name and embeds the US-English TeX
patterns (~88 KB). Patterns for the ~70 other supported languages are not
embedded; you download a precompiled `.bincode` file and load it yourself. The
compile-time sibling crate `textwrap-macros` offers procedural macros for
wrapping string literals known at build time.

## Production Notes

**Features are the whole story — audit them.** A default build enables `smawk`,
`unicode-linebreak`, and `unicode-width`. If binary size or compile time
matters, disabling `unicode-linebreak` and `smawk` (`default-features = false`)
drops the biggest transitive costs, at the price of greedy wrapping and
ASCII-only break points. Conversely, forgetting to enable `unicode-width` will
silently misalign any CJK or emoji output.

**The clap coupling era is instructive.** For years textwrap was an indirect
dependency of a large fraction of Rust CLIs because `clap` 2.x used it to wrap
help text. That made textwrap's `unicode-width` version bumps ripple across the
ecosystem, and clap eventually removed the dependency to control its own
formatting[^3]. If you see textwrap in an old lockfile you did not add
directly, this is usually why.

**Correctness is width-model dependent.** Textwrap wraps to a *column* count,
not a pixel or byte count. Terminals that render emoji or flags as a single
cell versus two will disagree with `unicode-width` in edge cases; grapheme
clusters formed by ZWJ sequences are a known source of off-by-one rendering.
This is a Unicode-ecosystem limitation, not a textwrap bug, but it surfaces as
"my wrapped table is one column off."

**Performance.** OptimalFit is linear in words and fast enough for terminal
output, but it allocates and does more work than FirstFit. For hot loops that
wrap millions of short strings, benchmark both algorithms; for anything
human-facing, the difference is imperceptible and OptimalFit's output is nicer.

**API churn is behind you.** The 0.13 rewrite (2020) reorganized the crate
around `Options` and the `core` module and is the last large breaking change;
the 0.16 line has been stable since 2022. Upgrades within 0.16 are routine. The
crate is pre-1.0 by convention, not because it is unstable.

## When to Use / When Not

**Use when:**
- You need correct, Unicode-aware wrapping of terminal or PDF/canvas text.
- You want optimal-fit (balanced) line breaks without implementing Knuth-Plass.
- You want fine control over indentation, hyphenation, or break behavior.
- You want to pay only for the features you enable.

**Avoid when:**
- You need full paragraph *layout* with fonts, alignment, and shaping — reach
  for a text-shaping/layout stack, not a line wrapper.
- Your CLI framework already formats help text (modern `clap` does its own).
- You want zero dependencies and only wrap ASCII — a hand-rolled greedy
  splitter is a few lines and avoids the feature tree entirely.

## Alternatives

- unicode-rs/unicode-width — the width primitive textwrap builds on; use directly when you only need column measurement, not wrapping.
- clap-rs/clap — if the reason you wanted textwrap was `--help` formatting, clap now handles that itself; use it instead of wrapping by hand.
- console-rs/console — use when you want terminal styling, truncation, and measurement together rather than paragraph wrapping.
- rust-cli/textwrap-alternatives (comfy-table / prettytable) — use a table crate when the real goal is column layout, not free-text wrapping.
- Python's `textwrap` (stdlib) — the conceptual ancestor; reference it when porting behavior, not as a Rust dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016-12 | Initial release; simple greedy word wrapping. |
| 0.11 | 2018 | Widely adopted as an indirect dependency via clap 2.x. |
| 0.13 | 2020-12 | Major rewrite: `Options` builder, `core` module, SMAWK optimal-fit. |
| 0.14–0.15 | 2021 | Feature modularization; Unicode line-break and width behind flags. |
| 0.16 | 2022 | Dependency slimming; current stable line. |

## References

[^1]: SMAWK / optimal-fit line breaking follows the Knuth-Plass model used by TeX; textwrap implements it via the author's `smawk` crate. https://docs.rs/textwrap/latest/textwrap/wrap_algorithms/
[^2]: Cargo feature list and their binary-size impact. https://docs.rs/textwrap/#cargo-features
[^3]: clap removed its textwrap dependency to own help-text formatting. https://github.com/clap-rs/clap/blob/master/CHANGELOG.md

## Tags

rust, text-wrapping, cli, unicode, hyphenation, terminal, formatting, line-breaking, smawk, text-layout
