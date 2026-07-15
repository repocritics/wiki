# zhiburt/tabled

> A Rust library for rendering structs and enums as text tables, driven by a derive macro and a composable settings model.

[GitHub repo](https://github.com/zhiburt/tabled) ·
[crates.io](https://crates.io/crates/tabled) ·
[docs.rs](https://docs.rs/tabled) ·
[License: MIT](https://github.com/zhiburt/tabled/blob/master/LICENSE-MIT)

## Overview

tabled is a table-formatting library for the terminal. Its distinguishing idea is that a table is derived from your data types: put `#[derive(Tabled)]` on a struct or enum and `Table::new(rows)` produces a bordered ASCII/Unicode table, with the struct field names becoming column headers[^1]. This is the typed path. A runtime `Builder` path also exists for when the schema is not known at compile time.

The library is not just a printer — it is a composition system. Every visual concern (borders, alignment, padding, spans, truncation, color) is expressed as a `Setting` applied through `Table::with(..)` or `Table::modify(target, ..)`, where the target is an `Object` (rows, columns, a single cell, a segment). This makes tabled expressive but gives it a large surface area: the README's own table of contents lists dozens of setting types, and the correct way to do something is often non-obvious. The defining tension is exactly this — it is more capable than the alternatives, and correspondingly heavier to learn and to compile.

tabled is pre-1.0 and has been for its entire life since 2020[^2]. It is actively maintained (recent commits as of mid-2026) with a wide feature set, but the 0.x versioning is not cosmetic: minor releases have carried breaking API changes, and a major restructure moved most types under a `settings` module in earlier releases. Treat version pins and upgrades as real work.

## Getting Started

```bash
cargo add tabled
```

```rust
use tabled::{Table, Tabled};
use tabled::settings::Style;

#[derive(Tabled)]
struct Language {
    name: &'static str,
    designed_by: &'static str,
    invented_year: usize,
}

fn main() {
    let languages = vec![
        Language { name: "C",    designed_by: "Dennis Ritchie", invented_year: 1972 },
        Language { name: "Rust", designed_by: "Graydon Hoare",  invented_year: 2010 },
    ];

    let mut table = Table::new(languages);
    table.with(Style::modern());
    println!("{table}");
}
```

For data whose shape is only known at runtime, build rows imperatively:

```rust
use tabled::builder::Builder;
use tabled::settings::Style;

let mut b = Builder::default();
b.push_record(["name", "year"]);
b.push_record(["C", "1972"]);
let mut table = b.build();
table.with(Style::psql());
```

## Architecture / How It Works

The repository is a Cargo workspace of several crates, and the split matters for understanding both capability and compile cost:

- **`tabled`** — the high-level API: the `Tabled` trait, `Table`, `Builder`, and the `settings` module (styles, objects, alignment, width, spans, color).
- **`tabled_derive`** — the proc-macro crate implementing `#[derive(Tabled)]`. It generates the `fields()`/`headers()` implementation, and supports attributes like `#[tabled(rename = ..)]`, `#[tabled(skip)]`, `#[tabled(order = ..)]`, and `#[tabled(inline)]` for flattening nested structs.
- **`papergrid`** — the low-level grid engine. It owns the actual layout math: measuring each cell, computing column widths, and drawing borders. tabled is essentially an ergonomic front-end over papergrid, and papergrid can be used directly if you want the grid without the settings sugar.

The `Tabled` trait reduces a value to a vector of stringified fields plus a header list. `Table` buffers all of those records into an in-memory grid, then papergrid scans every cell to compute per-column widths before rendering — so a full `Table` is inherently a two-pass, whole-dataset operation.

Two axes are worth internalizing. First, **`Style` is compile-time, `Theme` is runtime**: `Style::modern()`, `Style::psql()`, etc. are `const`-friendly and cheap, whereas `Theme` exists for cases where the look must be decided dynamically. Second, tabled ships **multiple table types for different cost profiles**: `Table` (full in-memory grid), `IterTable` and `CompactTable` (stream or render with minimal/no allocation, suitable for `no_std` and large or unbounded data), `PoolTable` (arbitrary nested layouts), `ExtendedTable` (vertical key/value output), and `Table::kv`. Choosing the wrong one is the most common source of memory and performance surprises.

## Production Notes

- **Pre-1.0, breaking minor releases.** Pin an exact version. Upgrades across `0.x` bumps have historically required code changes; the migration to the `settings`-module layout in particular broke a lot of downstream code. Budget for it.
- **`Table` holds everything.** Building a full `Table` allocates the entire record set and width-scans every cell — O(rows x cols) time and memory. For large, streamed, or `no_std` workloads use `IterTable`/`CompactTable`, which the README explicitly points to for low-footprint rendering.
- **Color needs the right feature.** ANSI-colored strings must be handled by width-aware parsing or every escape sequence is counted as visible width and columns misalign. Enable the ANSI/color feature; do not feed raw colored strings into a build without it.
- **Unicode and emoji width is best-effort.** Width is computed via unicode-width. East-Asian wide characters generally work; emoji with ZWJ sequences, skin-tone modifiers, or variation selectors can still misalign, and the README carries an explicit Emoji caveat. Verify visually if your data contains them.
- **MSRV is not frozen.** The minimum supported Rust version is bumped as needed and is not treated as a stability guarantee — this can break CI pinned to older toolchains.
- **Compile time and binary size.** The derive macro plus heavily generic settings monomorphize per call site; the settings API's flexibility costs build time. Trim features (disable `derive`, `ansi`, `macros`) for leaner builds, and consider `CompactTable` on constrained targets.
- **Width settings operate on rendered content.** `Truncate`, `Wrap`, and justification act on already-measured cells and depend on the width feature; combining them with color and spans requires care to get correct results.

## When to Use / When Not

**Use when:**
- You have typed data (structs/enums) and want headers and columns derived for free via `#[derive(Tabled)]`.
- You need fine control over borders, spans, per-cell styling, merged cells, or unusual layouts that simpler libraries cannot express.
- You want one library that covers both compile-time-fixed styling and runtime-dynamic theming, plus multiple rendering strategies (full, streaming, nested, key/value).

**Avoid when:**
- You want a table that automatically fits itself to the terminal width with minimal configuration — comfy-table targets that use case more directly.
- You need only simple aligned columns without borders — an elastic-tabstop formatter is lighter.
- Build time, binary size, or a stable pre-1.0 API are hard constraints; tabled's flexibility and 0.x churn cut against all three.

## Alternatives

- nukesor/comfy-table — runtime-oriented tables with automatic width arrangement to the terminal; use it when dynamic terminal fitting matters more than derive-from-struct ergonomics.
- phsym/prettytable-rs — the classic macro-driven prettytable port; use it when you want the familiar `prettytable` API and are content with a more lightly maintained crate.
- BurntSushi/tabwriter — elastic-tabstop column alignment with no borders; use it when you only need text columns lined up, not a drawn table.
- RyanBluth/term-table-rs — a smaller borders-and-cells table crate; use it when you want basic tables without tabled's settings surface and compile cost.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-02-25 | Repository created; first `0.x` releases on crates.io[^2]. |
| 0.x (settings era) | ~2022–2023 | Major restructure into the `settings`/`object` model and the `papergrid` backend split[^3]. |
| 0.x (current) | 2026-06 | Actively maintained; still pre-1.0. Latest commit mid-2026[^4]. |

## References

[^1]: tabled README — usage, derive macro, and settings overview. https://github.com/zhiburt/tabled#readme
[^2]: GitHub repository metadata (created 2020-02-25). https://github.com/zhiburt/tabled
[^3]: docs.rs API reference for the `settings` module and `papergrid`. https://docs.rs/tabled
[^4]: GitHub repository — commit/activity history (last push 2026-06-23). https://github.com/zhiburt/tabled/commits/master

## Tags

rust, cli, table, pretty-print, terminal, text-formatting, derive-macro, ascii, tui, library
