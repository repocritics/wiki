# BurntSushi/aho-corasick

> A Rust library for searching many string patterns at once in linear time, with configurable match semantics and optional SIMD acceleration.

[GitHub repo](https://github.com/BurntSushi/aho-corasick) ·
[Docs](https://docs.rs/aho-corasick) ·
[License: MIT OR Unlicense](https://github.com/BurntSushi/aho-corasick/blob/master/LICENSE-MIT)

## Overview

`aho-corasick` implements the Aho-Corasick algorithm[^1]: it compiles a set of
string patterns into a finite state machine that finds every occurrence of any
pattern in a single left-to-right pass over the input, in time proportional to
the input length regardless of how many patterns are being searched. It is
written by Andrew Gallant (BurntSushi), who also authors `ripgrep`, `regex`, and
`memchr`, and it is one of the load-bearing crates in that text-search stack —
the `regex` crate uses it internally to accelerate searches that reduce to a set
of literal alternatives[^2].

The library's distinguishing feature is not the algorithm — that is textbook —
but the amount of practical engineering wrapped around it. It exposes multiple
in-memory automaton representations that trade memory for speed, three different
match semantics (which is where the textbook algorithm quietly disagrees with
what most programmers expect), and a separate SIMD-based `packed` search path
that outperforms the automaton entirely when the number of patterns is small.
For a crate whose public surface looks like "give me strings, get back matches,"
a surprising amount of the design is about giving the caller control over those
tradeoffs without paying for them when they are not needed.

The central tension is that the naive Aho-Corasick match report — "report every
match as soon as it is seen" — does not match how programmers (or Perl-like
regex engines) think about matching. Searching `["Samwise", "Sam"]` against
`Samwise` yields `Sam` under the standard formulation but `Samwise` under
leftmost-first semantics. This crate treats that choice as a first-class, cost-free
build option rather than a post-processing step[^3].

## Getting Started

```bash
cargo add aho-corasick
```

```rust
use aho_corasick::{AhoCorasick, PatternID};

let patterns = &["apple", "maple", "Snapple"];
let haystack = "Nobody likes maple in their apple flavored Snapple.";

let ac = AhoCorasick::new(patterns).unwrap();
let mut matches = vec![];
for mat in ac.find_iter(haystack) {
    matches.push((mat.pattern(), mat.start(), mat.end()));
}
assert_eq!(matches, vec![
    (PatternID::must(1), 13, 18),
    (PatternID::must(0), 28, 33),
    (PatternID::must(2), 43, 50),
]);
```

Match semantics and case sensitivity are set on the builder:

```rust
use aho_corasick::{AhoCorasick, MatchKind};

let ac = AhoCorasick::builder()
    .match_kind(MatchKind::LeftmostFirst)   // Perl-like: prefer earliest-added, longest-reaching
    .ascii_case_insensitive(true)
    .build(&["Samwise", "Sam"])
    .unwrap();
let mat = ac.find("Samwise").unwrap();
assert_eq!("Samwise", &"Samwise"[mat.start()..mat.end()]);
```

## Architecture / How It Works

Internally the crate does not have one automaton — it has three, exposed through
`AhoCorasickKind`:

- **Noncontiguous NFA** — a trie with failure transitions stored as a sparse
  per-state structure. Cheapest to build, largest per-state pointer chasing,
  slowest to search.
- **Contiguous NFA** — the same NFA packed into a single contiguous allocation
  with a compact state encoding. Better cache behavior; the usual default for
  moderate pattern counts.
- **DFA** — failure transitions fully precomputed into a dense transition table,
  so each byte is a single table lookup with no failure-link walking. Fastest
  search, but memory grows with `states × 256`, which can be large.

`AhoCorasick::new` picks a representation heuristically from the pattern set;
`AhoCorasickBuilder` lets you force one. This is the memory-versus-throughput
knob, and it is the main thing to understand before tuning.

Separately, the `packed` module implements **Teddy**, a SIMD multi-substring
algorithm derived from Intel's Hyperscan. Teddy does not build an automaton at
all; it uses vectorized candidate filtering and is dramatically faster than any
of the automatons — but only for a small number of relatively short patterns.
`AhoCorasick` can use a packed searcher as a prefilter automatically. Underneath,
single-byte scanning falls through to the `memchr` crate's SIMD routines. On
targets without SIMD, or for large pattern sets, everything degrades cleanly back
to the automaton path.

Match semantics (`MatchKind::Standard`, `LeftmostFirst`, `LeftmostLongest`) are
baked into the compiled machine, not applied afterward, so leftmost-first search
costs the same as standard search. `Standard` supports overlapping matches;
the leftmost variants do not, because "leftmost" and "all overlapping" are
mutually exclusive concepts.

## Production Notes

- **DFA memory is the real footgun.** For large or numerous patterns, forcing
  `AhoCorasickKind::DFA` can produce a transition table that is orders of
  magnitude larger than the NFA. If build time or memory spikes unexpectedly,
  this is almost always why. Let `new` choose unless you have measured.

- **Overlapping matches require `MatchKind::Standard`.** `find_overlapping_iter`
  and related methods panic (or are unavailable) under leftmost semantics. Pick
  the match kind up front based on whether you need overlaps; you cannot mix.

- **Build cost is paid once, search is cheap.** Construction is the expensive
  phase (especially for the DFA). The intended usage is build-once, search-many.
  Rebuilding an `AhoCorasick` per query throws away the entire value proposition.

- **`&str` vs `&[u8]`.** The API operates on bytes and returns byte offsets. When
  searching UTF-8 text, offsets are valid `str` indices only because patterns are
  matched on byte boundaries; if you feed arbitrary binary and slice a `&str`
  with the results you can hit a non-char-boundary panic. Search bytes, slice
  bytes.

- **1.0 was a full rewrite.** The pre-1.0 (`0.7.x`) API is materially different —
  types were renamed, the automaton kinds were reorganized, and `MatchKind`
  handling changed. Upgrading across the 1.0 boundary is a real migration, not a
  version bump, though the crate has been API-stable since[^4].

- **Stream search is not zero-copy.** `stream_replace_all` and the streaming
  search APIs operate over `io::Read`/`io::Write` without buffering the whole
  input, but they carry overhead versus the in-memory path and are the right tool
  only when the haystack genuinely does not fit in memory.

## When to Use / When Not

**Use when:**
- You are searching for many fixed string patterns simultaneously and want linear
  time regardless of pattern count.
- You need control over match semantics (leftmost-first / leftmost-longest /
  overlapping) that a naive multi-search does not give you.
- You are doing search-and-replace over large or streaming text with a fixed
  dictionary of terms.
- You want a dependency-light, well-audited building block that the `regex` crate
  itself trusts.

**Avoid when:**
- You have a single pattern — use `memchr::memmem` or `str::find`; Aho-Corasick is
  overkill.
- Your patterns are regular expressions, not fixed strings — use `regex` (which
  will use this crate for you where it helps).
- You need approximate/fuzzy matching or wildcards — this is exact-match only.
- Patterns change on every query — construction cost dominates and you lose the
  benefit.

## Alternatives

- rust-lang/regex — use when your patterns are real regexes, not fixed literals;
  it layers this crate underneath for the literal-set case.
- BurntSushi/memchr — use for single-substring or single-byte SIMD search; lighter
  and faster when you are not matching a *set*.
- daac-tools/daachorse — use when you want a double-array Aho-Corasick trie in Rust
  optimized for compact automaton size and fast construction.
- intel/hyperscan — use for very large-scale, regex-and-literal streaming matching
  in C/C++ where you can accept a heavier dependency; the origin of the Teddy idea.
- G-Research/ahocorasick_rs — use from Python; it is a binding over this crate.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.5–0.7.x | 2016–2022 | Long-lived pre-1.0 line; earlier type layout and API. |
| 1.0.0 | 2023 | Full rewrite: `AhoCorasickKind` NFA/DFA representations, reworked `MatchKind`, packed/Teddy integration cleaned up[^4]. |
| 1.1.x | 2023–2024 | Follow-on releases on the stable 1.x API; current line. |

Exact 1.x patch dates and version numbers were not verified against crates.io
at write time; treat the milestone dates above as approximate.

## References

[^1]: Aho, Alfred V.; Corasick, Margaret J., "Efficient string matching: an aid to bibliographic search" (1975). Overview: https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm
[^2]: `regex` crate — uses `aho-corasick` for literal/prefilter acceleration. https://github.com/rust-lang/regex
[^3]: aho-corasick README, "finding the leftmost first match." https://github.com/BurntSushi/aho-corasick
[^4]: `aho-corasick` on docs.rs — current API reference (`AhoCorasick`, `AhoCorasickKind`, `MatchKind`, `packed`). https://docs.rs/aho-corasick

## Tags

rust, aho-corasick, string-matching, finite-state-machine, text-search, simd, multi-pattern-search, substring-matching, text-processing, algorithms
