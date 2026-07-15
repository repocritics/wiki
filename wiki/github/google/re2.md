# google/re2

> A regular expression engine that trades peak speed and exotic features for a hard guarantee: match time is linear in the input length.

[GitHub repo](https://github.com/google/re2) ·
[License: BSD-3-Clause](https://github.com/google/re2/blob/main/LICENSE)

## Overview

RE2 is a C++ regular expression library that has run in production at Google since 2006 and was open-sourced in 2010[^1]. Its defining stance is stated plainly in the README: *safety is the primary goal*. RE2 will accept patterns from untrusted users and still guarantee that matching completes in time linear in the length of the input, with memory bounded by a configurable budget. It does this by refusing to implement any regex construct that is only known to be solvable via backtracking — so **no backreferences and no look-around assertions**[^2].

That refusal is the whole tradeoff. A backtracking engine (PCRE, Perl, Python's `re`, Java's `java.util.regex`) is optimistic: it tries alternatives one at a time and is very fast when the first alternative usually matches, but can be driven to exponential blowup by a pathological pattern/input pair — the "regex denial of service" (ReDoS) class of vulnerabilities. RE2 is pessimistic: it evaluates all alternatives in lockstep via automata simulation, paying a constant-factor overhead on easy cases in exchange for never exploding. If your regexes come from a config file you wrote, a backtracker is often faster; if they come from user input, RE2 is the safe default and the reason it exists.

RE2 is the practical embodiment of Russ Cox's three-part series on regex theory[^3], which argues that the Thompson NFA construction — known since 1968 and quietly abandoned by most mainstream engines — is both simpler and asymptotically better than backtracking. The Go `regexp` package and the Rust `regex` crate share no code with RE2 but follow the same principles and syntax and offer the same linear-time guarantee[^2].

## Getting Started

RE2 requires a C++17 compiler and depends on Abseil[^4]. Install the dependency, then build with Make, CMake, or Bazel:

```bash
# Debian/Ubuntu
apt install libabsl-dev libgtest-dev libbenchmark-dev
# macOS
brew install abseil googletest google-benchmark pkg-config-wrapper

make && make test && make install
```

Minimal C++ usage:

```cpp
#include <re2/re2.h>
#include <cassert>
#include <string>

int main() {
  // Compile once, reuse across calls.
  RE2 re("(\\w+):(\\d+)");
  assert(re.ok());               // check re.error() if false

  std::string user; int port;
  assert(RE2::FullMatch("ruby:1234", re, &user, &port));
  assert(user == "ruby" && port == 1234);

  // PartialMatch scans for a substring; FullMatch anchors both ends.
  assert(RE2::PartialMatch("see ruby:1234 here", re));
}
```

The official Python wrapper is published as `google-re2` on PyPI — note the bare `re2` package is a different, unmaintained project[^2].

## Architecture / How It Works

RE2 is not one algorithm but a small pipeline feeding several interchangeable execution engines[^3]:

1. **Parse** — the pattern is parsed into a `Regexp` AST, then simplified (e.g. `a{2,4}` expanded, character classes normalized).
2. **Compile** — the AST is compiled to a `Prog`, a flat program of byte-level instructions (a "regex bytecode" for an NFA).
3. **Execute** — RE2 picks among multiple engines depending on what the caller asked for:
   - **DFA** — the fastest search path. Runs a lazy, on-the-fly deterministic automaton to answer *does it match* and *where does the match end/start*, but cannot record submatch boundaries. RE2 runs a forward DFA to find the match end and a reverse DFA over the compiled reverse program to find the start.
   - **OnePass NFA** — an optimized engine for patterns where the NFA is deterministic enough that submatches can be tracked in a single pass.
   - **BitState** — a bounded backtracker (bit-vector visited-set) for small programs and short inputs, used when it is cheaper than the general NFA.
   - **NFA (Pike VM)** — the general submatch engine: a Thompson NFA simulation that threads capture-group positions through all states in parallel. Linear time, higher constant factor.

The **lazy DFA** is the key performance trick and the key memory concern. A full DFA for a regex can have exponentially many states, so RE2 builds DFA states on demand and caches them. The cache lives within a memory budget (`RE2::Options::set_max_mem`, ~8 MB by default). When the cache fills, RE2 flushes it and keeps going — correctness is preserved, but a flush-thrash loop degrades the DFA back toward NFA-like speed. This is by design: bounded memory is part of the safety contract, not an afterthought.

Because there is no recursion in the engines and no unbounded stack use in the parser or compiler, RE2 avoids the stack-overflow failure mode that recursive backtrackers have on deeply nested patterns. Everything — parse, compile, execute — works inside the budget and fails gracefully when exhausted rather than crashing.

## Production Notes

- **`absl::string_view` submatches are non-owning.** Extracting a match into an `absl::string_view*` gives you a pointer into the *original* input buffer, not a copy. If the input string is freed or goes out of scope, the submatch dangles. Use `std::string*` targets when you need the match to outlive the input.
- **Compile once.** `RE2::FullMatch("...", "pattern", ...)` recompiles the pattern on every call. In hot paths, construct a persistent `RE2` object (or a cached pool of them) and reuse it. Compilation is not free.
- **The memory budget is a real knob.** The default `max_mem` (~8 MB) is per-`RE2`-object and covers the compiled program *and* the DFA cache. Complex patterns over large alphabets (especially with `\p{...}` Unicode classes) can exhaust it and fall back to the slower NFA. Raise `max_mem` for known-heavy patterns; lower it if you are compiling many patterns from untrusted input and need to bound total footprint.
- **No implicit normalization.** RE2 matches on Unicode code points and does no Unicode normalization. `/ü/` (U+00FC) will not match a decomposed `u`+combining-diaeresis input. Normalize both pattern and input yourself if you need canonical-equivalence matching.
- **Abseil coupling is a build cost.** Since the move to C++17 + Abseil, RE2 pulls in a large, ABI-sensitive dependency. Abseil has no stable ABI guarantee across releases, so RE2 and everything else linking Abseil in your binary must be built against a compatible version — a recurring headache in mixed-source builds and distro packaging.
- **RE2 does not use GitHub pull requests.** The repo is a mirror; contributions go through Google's process described in the wiki, and issues are the discussion channel[^2]. Do not expect PRs to be reviewed on GitHub.
- **It is not always the fastest.** The README says so directly: RE2 does not aim to beat every engine on every input. For fixed, trusted patterns a good backtracker or a specialized matcher (Hyperscan for multi-pattern scanning) can win. RE2's value is the *worst-case* guarantee, not the best-case number.

## When to Use / When Not

**Use when:**
- Patterns come from users, config, or any untrusted source and ReDoS is a real threat.
- You need a hard upper bound on match time and memory (search infrastructure, log processing, WAFs, security scanners).
- You are matching against very large inputs where a backtracker's blowup risk is unacceptable.
- You want the same syntax/semantics as Go's `regexp` or Rust's `regex` in a C++ codebase.

**Avoid when:**
- You need backreferences (`\1`) or look-around (`(?=...)`, `(?<=...)`) — RE2 will never support them by design; use PCRE2.
- Patterns are fixed and trusted, and you have measured that a backtracker is faster for your workload.
- You cannot take on the Abseil + C++17 build dependency.
- You need multi-gigabit multi-pattern scanning — that is Hyperscan's specialty, not RE2's.

## Alternatives

- PCRE2 — full Perl feature set including backreferences and look-around; backtracking, so ReDoS-prone. Use when you genuinely need those features on trusted patterns.
- rust-lang/regex — same linear-time guarantees and syntax family as RE2, memory-safe, no Abseil; use when you are in Rust or want RE2's model without C++.
- intel/hyperscan — SIMD-accelerated multi-pattern streaming matcher; use when scanning input against thousands of patterns at line rate (IDS/DPI), not for single rich patterns.
- google/re2j — pure-Java port of RE2 following the same principles; use on the JVM when `java.util.regex`'s backtracking is a liability.
- PhilipHazel PCRE (original) — legacy; new projects should prefer PCRE2 if a backtracker is required.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2006 | In production inside Google's code search and other systems[^1]. |
| open-source | 2010-03 | Released as open source alongside Russ Cox's regex articles[^1][^3]. |
| GitHub mirror | 2014-08 | Repository mirrored to github.com/google/re2 (created 2014-08-18). |
| — | ~2022 | Migrated to require C++17 and the Abseil library; adopted `absl::string_view`[^4]. |
| date-tagged | ongoing | RE2 ships date-stamped release tags (e.g. `2024-xx-xx`) rather than semantic versions; last source push 2026-01. |

## References

[^1]: RE2 README — "RE2, a regular expression library." Notes production use at Google since 2006. https://github.com/google/re2/blob/main/README.md
[^2]: RE2 README — syntax, ports and wrappers, contribution process (no GitHub PRs). https://github.com/google/re2/blob/main/README.md
[^3]: Russ Cox, "Implementing Regular Expressions" — three-part series on NFA-based matching that RE2 embodies. https://swtch.com/~rsc/regexp/
[^4]: RE2 build instructions — C++17 compiler and Abseil dependency. https://github.com/google/re2/blob/main/README.md

## Tags

regex, cpp, regular-expressions, finite-automata, linear-time, redos-safe, text-processing, google, library, pattern-matching
