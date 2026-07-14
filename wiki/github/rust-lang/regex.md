# rust-lang/regex

> Rust's standard regular expression engine — finite-automata based, with a linear-time matching guarantee and no backreferences or look-around.

[GitHub repo](https://github.com/rust-lang/regex) ·
[Documentation](https://docs.rs/regex) ·
[License: MIT OR Apache-2.0](https://github.com/rust-lang/regex/blob/master/LICENSE-APACHE)

## Overview

`regex` is the de facto standard regular-expression library for Rust, maintained under the rust-lang organization and authored primarily by Andrew Gallant (BurntSushi)[^1]. It is not part of `std`, but it is the crate almost every Rust project reaches for, and it underlies well-known tools like ripgrep. The design lineage traces to Russ Cox's RE2: matching is implemented with finite automata rather than a backtracking engine, which is what gives every search a worst-case `O(m * n)` time bound (`m` proportional to regex length, `n` to haystack length)[^2].

The defining tradeoff follows directly from that choice. Because the engine never backtracks, it cannot support features that are only expressible with backtracking — most notably **backreferences** and **look-around assertions**. In exchange, it is immune to catastrophic backtracking (ReDoS): there is no pathological pattern/input pair that makes it hang. For untrusted patterns or untrusted input, this is the property that matters most, and it is the main reason to choose this crate over a PCRE-style engine.

The crate is a thin front-end over an internal stack: `regex-syntax` parses patterns into an AST and HIR, and `regex-automata` builds and runs the actual automata. A caller who wants only 99% of the functionality uses `regex::Regex`; a caller who wants to pick engines, inspect the HIR, or build a custom matcher drops down to those crates directly.

## Getting Started

```bash
cargo add regex
```

```rust
use regex::Regex;

fn main() {
    // Named captures; `(?x)` enables verbose mode with comments.
    let re = Regex::new(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})").unwrap();
    let caps = re.captures("2010-03-14").unwrap();
    assert_eq!("2010", &caps["year"]);
    assert_eq!("03", &caps["month"]);
}
```

Compile once, reuse many times — never build a `Regex` inside a hot loop:

```rust
use std::sync::LazyLock;
use regex::Regex;

fn is_ident(s: &str) -> bool {
    static RE: LazyLock<Regex> = LazyLock::new(|| Regex::new(r"^[A-Za-z_]\w*$").unwrap());
    RE.is_match(s)
}
```

## Architecture / How It Works

The public `regex::Regex` is mostly a wrapper around `meta::Regex` in the `regex-automata` crate[^3]. The interesting machinery lives underneath:

- **`regex-syntax`** — parses the pattern into an AST, then lowers it to an HIR (high-level intermediate representation) suitable for analysis. This crate is published separately and is useful on its own for anyone writing their own engine or doing static analysis of patterns.
- **`regex-automata`** — the meta engine. Rather than one algorithm, it holds several matching strategies and picks per search: a **lazy DFA / hybrid NFA-DFA** (builds DFA states on demand, capped by a memory budget), a **one-pass NFA** for patterns where captures are unambiguous, a **bounded backtracker** for small patterns/inputs, and the **PikeVM** as the general fallback that resolves capture groups in linear time.
- **Literal prefilters** — before running an automaton, the engine extracts required literals and uses `memchr` (SIMD memory scanning) and `aho-corasick` (multi-pattern literal search) to skip quickly to candidate positions. In practice this prefiltering is where much of the real-world speed comes from; a regex anchored by a rare literal is far faster than one that is not.

Two variants of the API exist. `regex::Regex` searches `&str` (guaranteed valid UTF-8). `regex::bytes::Regex` searches `&[u8]` and can match invalid-UTF-8 bytes when Unicode mode is disabled (`(?-u:.)` matches any non-newline byte). `RegexSet` matches many patterns against a haystack in a single scan and reports which ones matched, without telling you where.

Unicode support (`\w`, `\pL`, case folding, script classes) is backed by embedded Unicode tables. These tables are a significant fraction of the crate's compiled size and can be turned off via Cargo features when binary size matters more than Unicode correctness.

## Production Notes

**Compilation is the expensive step, not matching.** `Regex::new` takes microseconds to a few milliseconds depending on pattern size. Compiling the same pattern repeatedly — especially inside a loop or per-request handler — is the single most common performance mistake. Hoist compilation into a `LazyLock`/`OnceLock` static.

**No look-around, no backreferences — by design, permanently.** These are not missing features waiting to be added; they are incompatible with the linear-time model. If you genuinely need them in Rust, `fancy-regex` wraps this crate and layers a backtracker on top for the constructs this crate rejects, at the cost of the ReDoS guarantee.

**Binary size and compile time are tunable.** The default build pulls in full Unicode tables and multiple engines. Disabling `unicode` sub-features (`unicode-perl`, `unicode-case`, etc.) or building with `default-features = false, features = ["std"]` reduces the dependency tree to `regex-syntax` + `regex-automata` and shrinks the binary — at the cost of Unicode-aware character classes and, in some configurations, some runtime performance.

**The lazy DFA has a memory cap.** The hybrid engine builds DFA states on demand and evicts them under a configurable cache budget. Pathological patterns can thrash this cache and fall back to slower engines; the automata crate exposes knobs for the cache size if profiling shows this.

**MSRV moves in minor releases.** The minimum supported Rust version is 1.65.0[^4], and the policy explicitly permits raising the MSRV in a minor (`1.y`) bump. Pin your toolchain or your `regex` minor version if you support older compilers.

**The 1.9 rewrite (2023) reset the internals.** Version 1.9.0 replaced the old engine wholesale with the `regex-automata` meta engine[^5]. The public API stayed compatible, but performance characteristics, memory use, and edge-case behavior shifted. Benchmarks or timing assumptions taken before 1.9 do not carry forward. Benchmarking is now done out-of-tree via `rebar`.

## When to Use / When Not

**Use when:**
- You run regexes against untrusted input or untrusted patterns and cannot tolerate ReDoS.
- You want predictable, linear-time matching regardless of pattern complexity.
- You need Unicode-aware matching, `RegexSet` multi-pattern scans, or byte-level (`&[u8]`) searching.
- You are building Rust CLI tooling or services and want the ecosystem-standard engine.

**Avoid when:**
- You require backreferences or look-around — reach for `fancy-regex` instead.
- You are matching fixed strings — plain `str::contains`, `memchr`, or `aho-corasick` are faster and simpler.
- You need the smallest possible binary and can express the match another way — full Unicode tables are not free.
- You are compiling a fresh pattern per call and cannot cache it — the compile cost may dominate.

## Alternatives

- google/re2 — the C++ engine this crate's philosophy descends from; use it when you need the same linear-time guarantees outside Rust.
- rust-onig — Rust bindings to Oniguruma; use when you need PCRE-style features (look-around, backreferences) and accept a C dependency plus backtracking risk.
- fancy-regex — wraps this crate and adds backtracking for look-around/backreferences; use when those features are non-negotiable in pure Rust.
- BurntSushi/aho-corasick — multi-pattern literal matching; use instead of a `RegexSet` of plain strings.
- pcre2 (via bindings) — full PCRE2 feature set; use when compatibility with existing PCRE patterns matters more than the ReDoS guarantee.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2014-12 | Initial release under rust-lang[^1]. |
| 1.0.0 | 2018-05 | Stable 1.x API; SemVer stability commitment. |
| 1.3 | 2019 | `default-features = false` slimming, feature-gated Unicode tables. |
| 1.9.0 | 2023-07 | Full internal rewrite onto the `regex-automata` meta engine[^5]. |
| 1.10 | 2023 | Post-rewrite refinements; syntax and performance follow-ups. |
| 1.11 | 2024 | Continued maintenance on the meta-engine architecture. |

## References

[^1]: rust-lang/regex repository, maintained by Andrew Gallant (BurntSushi). https://github.com/rust-lang/regex
[^2]: Project README — linear `O(m * n)` worst-case guarantee, no look-around or backreferences. https://github.com/rust-lang/regex/blob/master/README.md
[^3]: `regex-automata` meta engine documentation. https://docs.rs/regex-automata/latest/regex_automata/meta/struct.Regex.html
[^4]: README, "Minimum Rust version policy" — MSRV 1.65.0, may rise in minor releases. https://github.com/rust-lang/regex/blob/master/README.md
[^5]: Andrew Gallant, "regex internals" write-up on the automata-based rewrite. https://burntsushi.net/regex-internals/

## Tags

rust, regex, regular-expressions, finite-automata, dfa, nfa, text-processing, pattern-matching, unicode, linear-time, library
