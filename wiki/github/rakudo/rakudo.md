# rakudo/rakudo

> The reference compiler for the Raku programming language (formerly Perl 6), targeting MoarVM, the JVM, and JavaScript.

[GitHub repo](https://github.com/rakudo/rakudo) ·
[Official website](https://rakudo.org) ·
[License: Artistic-2.0](https://github.com/rakudo/rakudo/blob/main/LICENSE)

## Overview

Rakudo is the primary — in practice, the only production — implementation of Raku, the language designed as Perl 6 over a fifteen-year gestation and renamed Raku in October 2019 to end the confusion with Perl 5[^1]. The repository dates to 2009; the first stable *language* release ("6.c Christmas") did not ship until Rakudo 2015.12[^2]. That gap is the project's origin story and its reputation problem in one sentence: the language that took too long to arrive, attached to one of the most feature-dense language designs ever completed — grammars as first-class objects, gradual typing, rationals by default, grapheme-level Unicode strings, and structured concurrency (`Promise`/`Supply`/`react`) all in core.

Rakudo is unusual among compilers in being largely self-hosted on a bootstrapping subset called NQP ("Not Quite Perl"), which itself compiles to three backends: MoarVM (the purpose-built VM and the only one that matters in production), the JVM, and JavaScript. The compiler ships monthly releases named `YYYY.MM`, decoupled from language versions (6.c, 6.d, 6.e-in-development) which are defined not by a document but by the Roast test suite — "the spec is the tests"[^3].

At 1,889 stars after 17 years, the audience is small; but the repo is genuinely active — pushed daily, monthly releases sustained for over a decade, and a multi-year compiler-frontend rewrite (RakuAST) in progress. This is a niche language with an unusually persistent core team, not an abandoned one.

## Getting Started

Via `rakubrew`, the version manager (recommended over distro packages, which lag)[^4]:

```bash
curl https://rakubrew.org/install-on-perl.sh | sh
rakubrew download        # latest MoarVM-backed release
zef install App::Mi6     # zef is the package manager
```

Or from source (MoarVM backend, downloads NQP + MoarVM automatically):

```bash
perl Configure.pl --gen-moar --gen-nqp --backends=moar
make && make install
```

A taste of what makes Raku distinct — grammars and rationals in core:

```raku
grammar Greeting {
    token TOP   { <hello> \s+ <name> }
    token hello { 'hello' | 'hi' }
    token name  { \w+ }
}

my $m = Greeting.parse("hello world");
say $m<name>;           # ｢world｣

say 0.1 + 0.2 == 0.3;   # True — decimals are Rat, not floating point
```

## Architecture / How It Works

Rakudo is a layer cake with a deliberate bootstrap:

1. **Raku frontend** — the Raku grammar and actions, written in a mix of NQP and Raku itself. Raku's own grammar engine parses Raku; user-defined grammars use the same machinery, which is why grammars are a first-class language feature rather than a bolted-on PEG library.
2. **NQP** — a restricted Raku subset with its own compiler, used to write the compiler toolchain. NQP compiles to all three backends; Rakudo rides on top of it.
3. **QAST** — the backend-independent AST that NQP lowers to, then translated to MoarVM bytecode, JVM bytecode, or JavaScript.
4. **MoarVM** — a VM built specifically for Raku's semantics[^5]: NFG strings (O(1) grapheme indexing via synthetic codepoints), a dynamic specializer ("spesh") that deoptimizes when assumptions break, a template JIT, and continuations to support Raku's gather/take and concurrency model.

The backends are not feature-equivalent. MoarVM is the default and the only one exercised by real users; the JVM backend builds but chronically lags on features and speed; the JavaScript backend is experimental. Treat "runs on JVM and JS" as an architecture claim, not a deployment recommendation.

**RakuAST** is the significant ongoing work: a rewrite of the compiler frontend around a documented, user-exposable AST (`use experimental :rakuast`), merged into the mainline in 2023[^6]. It is the prerequisite for macros (a Perl 6 promise never delivered), better compile-time introspection, and eventually language version 6.e. A growing percentage of Roast passes under the RakuAST-based grammar each month, but it is not yet the default parser.

## Production Notes

**Startup and precompilation.** Bare `raku -e ''` startup is roughly 100–150 ms — far better than the early years, still an order of magnitude slower than Perl 5 or Lua. Modules are precompiled to bytecode on first load into a precomp store; the *first* load of a heavy module graph can take many seconds, after which loads are fast. CI containers that discard the precomp cache pay the first-load cost every run — persist `~/.raku` or bake precompilation into the image.

**Runtime performance is workload-dependent.** Spesh + JIT make hot numeric and object-oriented loops respectable, but naive string/regex munging can still lose to Perl 5, and peak performance is well below JVM/V8-class runtimes. Benchmark your actual workload; do not extrapolate from microbenchmarks in either direction.

**Monthly releases, ecosystem regressions.** Compiler releases are monthly and occasionally regress modules; the release team runs Blin (an ecosystem-wide regression tester) before tagging, which catches most but not all breakage. Pinning a known-good Rakudo version per project (easy with rakubrew) is standard practice.

**Small ecosystem.** raku.land lists a few thousand distributions[^7] — many one-maintainer, some unmaintained. Expect to write bindings yourself; `NativeCall` (C FFI in core) is genuinely good and is the standard escape hatch.

**Concurrency footguns.** `start`/`Promise`/`Supply` are pleasant, but `hyper`/`race` auto-parallelization has overhead that only pays off for coarse work units, and unhandled exceptions inside `Supply` taps can vanish without `QUIT` handling.

**865 open issues** reflects an old repo used as long-term memory (design discussions, dormant edge cases), not triage collapse — but it does mean a bug you hit may be known, documented, and years from a fix. Core-team attention is concentrated on RakuAST.

## When to Use / When Not

**Use when:**
- You need serious parsing: Raku grammars are the best in-language parsing facility shipped in any general-purpose language core.
- Text processing with correct Unicode by default (grapheme-based strings, no surrogate surprises).
- You want an expressive multi-paradigm scripting language for personal tooling and can accept the ecosystem size.
- You are a Perl 5 shop exploring succession paths and want maximal conceptual continuity.

**Avoid when:**
- You need a large third-party library ecosystem — Python, Ruby, or Node beat Raku by two to three orders of magnitude here.
- Latency-sensitive CLIs invoked at high frequency (startup cost) or hot paths needing peak throughput.
- Hiring matters: the pool of Raku-experienced developers is very small.
- You need the JVM or JS backends specifically — they exist but are not production-grade.

## Alternatives

- perl/perl5 — use instead when you need the mature CPAN ecosystem and battle-tested deployment; Raku is a different language, not an upgrade path.
- python/cpython — use instead when library availability and hiring outweigh language expressiveness.
- ruby/ruby — closest mainstream language in spirit (expressive, multi-paradigm, developer-happiness-oriented) with a vastly larger ecosystem.
- elixir-lang/elixir — use instead when the concurrency model is your primary draw and you want it on a production-hardened runtime (BEAM).
- crystal-lang/crystal — use instead when you want expressive syntax with static compilation and native-binary performance.

## History

| Version | Date | Notes |
|---------|------|-------|
| dev release #1 | 2009-01 | First numbered monthly development release, Parrot VM backend. |
| Rakudo Star 2010.07 | 2010-07 | First "useful, usable" distribution bundle (compiler + modules + docs). |
| JVM backend | 2013 | NQP and Rakudo ported to the JVM. |
| MoarVM backend | 2014 | MoarVM becomes the primary backend; Parrot support later removed. |
| 2015.12 | 2015-12-25 | Language version 6.c "Christmas" — first stable Perl 6 release[^2]. |
| 2018.12 | 2018-12 | Language version 6.d "Diwali" becomes default[^8]. |
| 2019-10 | 2019-10 | Perl 6 renamed Raku; `raku` becomes the executable name[^1]. |
| 2023.02 | 2023-02 | RakuAST merged into mainline as `use experimental :rakuast`[^6]. |
| ongoing | monthly | `YYYY.MM` compiler releases; 6.e available via `use v6.e.PREVIEW`. |

## References

[^1]: "Path to Raku", Raku problem-solving repo — the October 2019 rename decision. https://github.com/Raku/problem-solving/blob/master/solutions/language/Path-to-Raku.md
[^2]: Rakudo 2015.12 release announcement (6.c "Christmas"). https://github.com/rakudo/rakudo/blob/main/docs/announce/2015.12.md
[^3]: Roast — the official Raku specification test suite. https://github.com/Raku/roast
[^4]: rakubrew — Raku version manager. https://rakubrew.org/
[^5]: MoarVM — a VM built for Rakudo. https://moarvm.org/
[^6]: Rakudo 2023.02 release announcement (RakuAST). https://github.com/rakudo/rakudo/blob/main/docs/announce/2023.02.md
[^7]: raku.land — the Raku module ecosystem index. https://raku.land/
[^8]: Rakudo 2018.12 release announcement (6.d "Diwali"). https://github.com/rakudo/rakudo/blob/main/docs/announce/2018.12.md

## Tags

raku, perl6, compiler, programming-language, moarvm, nqp, grammars, gradual-typing, concurrency, unicode
