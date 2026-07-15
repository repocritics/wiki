# PCRE2Project/pcre2

> The Perl-compatible regex engine embedded in Git, PHP, Apache, Nginx, and countless other tools — feature-rich, fast with JIT, and exponential-time by design.

[GitHub repo](https://github.com/PCRE2Project/pcre2) ·
[Official website](https://pcre2project.github.io/pcre2/) ·
License: BSD-3-Clause with PCRE2 exception[^1]

## Overview

PCRE2 is a C library of functions implementing regular-expression pattern
matching using syntax and semantics close to Perl 5[^2]. It is the second-major
API generation of PCRE, a project originally written by Philip Hazel in 1997.
PCRE2 (the `10.x` series) shipped in 2015 with an entirely new, incompatible API
and is not source-compatible with the legacy PCRE (`8.x`), which reached
end-of-life at release 8.45 in 2021[^3]. New projects should always target
PCRE2; "PCRE" in modern dependency lists almost always means this library.

Its reach is the reason it matters more than its star count suggests: PCRE2 is
the regex backend for PHP's `preg_*` functions, R, Git (`git grep -P`), Apache
HTTPD, Nginx, and it is bundled into products like Safari and Excel. It is
self-contained portable C99 with no runtime dependencies, which is why it ends
up vendored into so many build systems.

The defining tension is capability versus safety. The default engine is a
depth-first backtracking matcher that supports backreferences, lookaround, and
recursion — features a linear-time automaton cannot express — but pays for them
with worst-case exponential match time (catastrophic backtracking / ReDoS). PCRE2
mitigates this with configurable match limits rather than by changing the
algorithm; treating untrusted patterns as safe is the most common way teams get
hurt by it.

## Getting Started

Most users consume PCRE2 through a distro package (`libpcre2-dev` on Debian/Ubuntu,
`pcre2` in Homebrew/vcpkg) rather than building it. To build from source:

```bash
git clone https://github.com/PCRE2Project/pcre2.git && cd pcre2
git submodule update --init      # required for JIT (sljit lives in a submodule)
cmake -B build -DPCRE2_SUPPORT_JIT=ON .
cmake --build build/
```

```c
#define PCRE2_CODE_UNIT_WIDTH 8   /* MUST be set before including the header */
#include <pcre2.h>
#include <string.h>

int errnum; PCRE2_SIZE erroff;
pcre2_code *re = pcre2_compile(
    (PCRE2_SPTR)"c.t", PCRE2_ZERO_TERMINATED, 0, &errnum, &erroff, NULL);

pcre2_match_data *md = pcre2_match_data_create_from_pattern(re, NULL);
const char *subject = "dogs and cats";
int rc = pcre2_match(re, (PCRE2_SPTR)subject, strlen(subject),
                     0, 0, md, NULL);   /* rc > 0 on match */

pcre2_match_data_free(md);
pcre2_code_free(re);
```

Link against the width-specific library: `-lpcre2-8`, `-lpcre2-16`, or
`-lpcre2-32`. The compiled `pcre2_code` is read-only and shareable across
threads; each thread needs its own `pcre2_match_data`.

## Architecture / How It Works

PCRE2 compiles a pattern once into an opaque byte-coded `pcre2_code`, then runs
it through one of three separate matching engines:

1. **Interpreted backtracking** (`pcre2_match`) — the default. A depth-first
   tree search over the compiled bytecode. Supports the full feature set
   (backreferences, lookbehind/lookahead, atomic groups, recursion, `\K`,
   conditionals). Worst-case exponential; aborts when `match_limit` or the
   heap/depth limits are exceeded.
2. **JIT** (`pcre2_jit_compile` + `pcre2_match`) — compiles the pattern to
   native machine code via the bundled **sljit** backend (a git submodule,
   authored by Zoltán Herczeg). Typically several times faster than the
   interpreter; must be explicitly compiled in and invoked, and it allocates a
   separate JIT stack.
3. **DFA** (`pcre2_dfa_match`) — a simultaneous-state matcher with worst-case
   polynomial time. Slower in practice, cannot do backreferences, but finds the
   POSIX leftmost-longest match and supports partial/streaming matching.

The library is built three times over for 8-, 16-, and 32-bit code units from a
single source tree using macro-driven width parameterization; `PCRE2_CODE_UNIT_WIDTH`
selects which set of symbols the header exposes. UTF-8/16/32 and EBCDIC are
supported, along with a large but manually-updated Unicode property database.

A POSIX-compatible `<pcre2posix.h>` / `regcomp`/`regexec` shim exists for drop-in
replacement, but it cannot pass PCRE2 options; the native API is recommended.

## Production Notes

- **ReDoS is the headline risk.** The default engine backtracks; adversarial
  patterns or adversarial input against innocent patterns can hang. Always set
  `pcre2_set_match_limit` and `pcre2_set_depth_limit` (older: recursion limit)
  when either the pattern or the subject is untrusted. There is no way to make
  backtracking linear-time — if you need guaranteed-linear behavior on hostile
  input, use RE2 or Rust `regex`, not PCRE2.
- **JIT and W^X.** The JIT needs memory that is both writable and executable,
  which conflicts with hardened environments: iOS, SELinux with `execmem`
  denied, OpenBSD's default `mprotect` restrictions, and some container
  seccomp profiles will block it. PCRE2 supports a two-step "executable
  allocator" workaround on some platforms, but the safe fallback is to disable
  JIT and accept the interpreter's speed.
- **The header-width footgun.** Forgetting to define `PCRE2_CODE_UNIT_WIDTH`
  before `#include <pcre2.h>` produces confusing compile errors; defining it to
  one width but linking a different `pcre2-N` library is worse — it links but
  misbehaves. Pin one width per translation unit.
- **JIT stack sizing.** Deeply recursive or heavily-backtracking patterns can
  exhaust the default JIT stack and return `PCRE2_ERROR_JIT_STACKLIMIT`; you may
  need `pcre2_jit_stack_create` with a larger max and attach it via a match
  context.
- **Bundled vs system copy.** Because PCRE2 is so often vendored, projects can
  silently ship an old, CVE-affected copy. Several past CVEs were
  heap-overflows/out-of-bounds reads reachable from crafted patterns; audit
  which version your dependency actually compiles against, not what the OS
  package reports.
- **Unicode data is version-locked.** The Unicode property tables are baked in
  at the PCRE2 release; matching `\p{...}` against very new scripts/emoji
  requires a recent PCRE2, not just a recent OS.

## When to Use / When Not

**Use when:**
- You need Perl-syntax features an automaton cannot do: backreferences,
  lookbehind, recursion, conditionals, named subpatterns.
- You are embedding regex into C/C++ and want a mature, portable, permissively
  licensed dependency.
- You control the patterns (or can bound them with match limits) and want the
  JIT's throughput.

**Avoid when:**
- Patterns come from untrusted users and must run in bounded time — reach for a
  linear-time engine instead.
- You are in a managed language that already ships a regex library; only drop to
  PCRE2 if you specifically need Perl-compatibility it lacks.
- Your platform forbids JIT and you needed the JIT's speed to meet SLAs.

## Alternatives

- google/re2 — linear-time RE2 automaton, no backtracking or backreferences; use
  when patterns are untrusted and ReDoS is unacceptable.
- rust-lang/regex — linear-time, memory-safe Rust engine; use when your stack is
  Rust or you want RE2 guarantees with a friendlier API.
- VectorCamp/vectorscan (fork of Intel Hyperscan) — SIMD, multi-pattern,
  streaming matching; use for high-throughput scanning of many patterns at once.
- kkos/oniguruma — multi-encoding backtracking engine (Ruby's regex backend);
  use when you need Ruby/Oniguruma dialect compatibility specifically.
- PhilipHazel/pcre (legacy PCRE 8.x) — end-of-life; only for maintaining old code
  that cannot migrate to the new API.

## History

| Version | Date | Notes |
|---------|------|-------|
| PCRE 1.00 | 1997 | Original library by Philip Hazel[^2]. |
| PCRE2 10.00 | 2015-01 | New, incompatible API; start of the `10.x` line[^2]. |
| PCRE 8.45 | 2021 | Final legacy-PCRE release; `8.x` branch end-of-life[^3]. |
| — | 2021 | Maintenance moved to the community PCRE2Project GitHub org[^4]. |
| 10.42 | 2022-12 | Bug-fix and feature release in the `10.x` series. |
| 10.44 | 2024-06 | Continued `10.x` maintenance; JIT and Unicode updates. |

## References

[^1]: PCRE2 licence — BSD 3-clause with the PCRE2 exception. GitHub's API reports
`NOASSERTION` because the exception is non-standard, but the project ships a
BSD-3-Clause LICENCE.md. https://github.com/PCRE2Project/pcre2/blob/main/LICENCE.md
[^2]: PCRE2 project overview and documentation. https://pcre2project.github.io/pcre2/
[^3]: PCRE (legacy) end-of-life notice; final release 8.45. https://www.pcre.org/
[^4]: PCRE2 development repository (PCRE2Project org). https://github.com/PCRE2Project/pcre2

## Tags

c, regex, regular-expressions, pattern-matching, perl-compatible, jit, unicode, library, text-processing, backtracking
