# racket/racket

> A general-purpose programming language in the Scheme/Lisp family, built around the idea that making new languages should be a normal part of programming.

[GitHub repo](https://github.com/racket/racket) ·
[Official website](https://racket-lang.org/) ·
License: LGPL-3.0 / Apache-2.0 / MIT (mixed; see repo)

## Overview

Racket is a mature, multi-paradigm language descended from PLT Scheme, which was renamed to Racket in 2010[^1]. It is a full ecosystem, not just a compiler: it ships with the DrRacket IDE, the `raco` command-line tool, a package manager and catalog, the Scribble documentation system, a contract system, first-class delimited continuations, and one of the most capable hygienic macro systems in production use. As of 2026 it has roughly 5.2k GitHub stars and is still actively developed, with commits landing within days — modest attention for a language of its depth and age.

The defining thesis is **language-oriented programming (LOP)**: the claim that the right response to a hard problem is often to build a small language tailored to it, and that a general-purpose language should make that cheap[^2]. The `#lang` line at the top of every Racket file names the language the rest of the file is written in — and that language is itself an ordinary Racket module you can write yourself. Typed Racket, the Scribble documentation language, the teaching languages, and Datalog implementations are all just `#lang`s built on the same substrate.

That power carries a cost that runs through everything below. Racket's community is small and concentrated in programming-languages research and education (it is the language of *How to Design Programs* and the Bootstrap curriculum). The library ecosystem is thin next to mainstream languages, hiring is hard, and raw performance is that of a dynamic language, not a systems one. Racket is chosen for what its macro and module system let you *build*, rarely for ecosystem breadth or throughput.

## Getting Started

Install a prebuilt distribution from the download site[^3], or via a package manager (`brew install --cask racket`, `apt install racket`, etc.). The distribution includes DrRacket and `raco`.

```racket
#lang racket

;; The first line selects the language. Everything below is Racket.
(define (greet name)
  (format "Hello, ~a" name))

(for ([name '("Tom" "Brad")])
  (displayln (greet name)))
```

Run it from the command line or open it in DrRacket:

```bash
racket hello.rkt
raco test hello.rkt     # run in-module tests
raco exe hello.rkt      # produce a standalone executable
```

A taste of the macro system — a new binding form defined in a few lines:

```racket
#lang racket

(define-syntax-rule (unless test body ...)
  (if test (void) (begin body ...)))

(unless (> 2 3)
  (displayln "2 is not greater than 3"))
```

## Architecture / How It Works

The core mechanism is the **module and macro expander**. A Racket source file is read into syntax objects, then expanded: macros rewrite syntax into a small core language before compilation. The expander is *hygienic* — identifiers introduced by a macro do not accidentally capture or get captured by user identifiers — and *phased*: compile-time code (macros, `require for-syntax`) and run-time code live in separate phases, so a module can compute the code of another module. `syntax-parse` provides the industrial-strength front end for writing macros with real error messages.

The **`#lang` reader** generalizes this. A language is a module that supplies a reader (how to parse the surface syntax into syntax objects) and an expander (what bindings the module gets). Because a language is just a library, non-parenthetical syntaxes, DSLs, and entirely new semantics are all reachable without touching the runtime. This is what "language-oriented programming" concretely means in the codebase.

Racket has had **two runtimes**, and the switch between them is the most consequential recent change:

- **Racket BC** ("before Chez") — the historical implementation, a C runtime executing Racket bytecode with a JIT. It carried decades of accumulated C.
- **Racket CS** ("Chez Scheme") — the current default since Racket 8.0 (2021)[^4]. Racket's macro expander and module system were reimplemented in Racket itself and made to sit on top of Chez Scheme as the backend compiler, which generates native machine code[^5]. CS improved compile-time metaprogramming performance and long-term maintainability; most user code runs unchanged, but FFI-heavy and embedding code sometimes needed adjustment across the transition.

Layered on this single substrate: **contracts** (higher-order runtime contracts with precise "blame" assignment across module boundaries), **delimited continuations** (`call/cc` plus `shift`/`reset`), **Typed Racket** (a gradually-typed `#lang` that inserts contracts at the typed/untyped boundary), **Scribble** (the docs are Racket programs), and **`raco`** (the swiss-army driver for building, testing, packaging, doc rendering, and executable creation).

## Production Notes

**Distribution is heavyweight.** `raco exe` plus `raco distribute` produces a standalone executable, but it embeds the runtime — expect tens of megabytes for a trivial program. There is no small static binary story comparable to Go or Rust; Racket assumes the whole ecosystem comes along.

**Performance is dynamic-language performance.** Racket CS closed much of the gap versus BC and is respectable for a GC'd dynamic language, but it does not compete with C, Rust, or the JVM's peak throughput, and it has GC pauses. Racket is picked for expressiveness, not for hot-path number-crunching.

**The Typed Racket boundary is a documented performance cliff.** Sound gradual typing enforces types at run time by wrapping values crossing the typed/untyped boundary in contracts. When typed and untyped modules are interleaved, that checking can dominate runtime — in the worst configurations by large multiples. This is not a bug but a fundamental cost of *sound* gradual typing, studied directly on Racket[^6]. Practical advice: keep typed/untyped boundaries coarse, or type whole subsystems rather than sprinkling annotations.

**The ecosystem is small and bus-factor sensitive.** The package catalog is real and `raco pkg` works well, but library breadth is a fraction of npm/PyPI/crates.io, many packages have a single maintainer, and you will more often write things yourself. Documentation, however, is a genuine strength — Scribble docs are unusually thorough and searchable.

**Community governance friction is part of the history.** The "Racket2" debate (2019) over whether to adopt conventional non-parenthetical syntax was contentious; it resolved into **Rhombus**, a new language built on the Racket runtime rather than a replacement of Racket itself. Rhombus is still pre-1.0 as of 2026 — treat it as forward-looking, not a migration target.

**Startup latency.** The runtime and expander take a noticeable moment to start compared to a C binary; for CLI tools invoked in tight loops this matters. Compiled `.zo` bytecode (`raco make`) and executables mitigate but do not eliminate it.

## When to Use / When Not

**Use when:**
- You are building a DSL, a compiler, or a language experiment — this is Racket's home turf, and nothing mainstream matches its macro/`#lang` machinery.
- You are teaching programming or doing programming-languages research.
- You want a batteries-included Scheme with an IDE, contracts, first-class continuations, and excellent docs out of the box.
- Metaprogramming and correctness matter more than raw speed or library count.

**Avoid when:**
- You need high-throughput production services or predictable low latency.
- You depend on a broad third-party library ecosystem (web frameworks, cloud SDKs, ML stacks).
- You need to hire from a large talent pool, or hand the codebase to a team unfamiliar with Lisp.
- You want small static binaries or mobile/browser front-end deployment.

## Alternatives

- clojure/clojure — a Lisp with a large production ecosystem via JVM/JS interop; use when you need libraries, hosting, and a hiring pool more than a custom-language toolkit.
- cisco/ChezScheme — the Scheme compiler Racket CS is built on; use directly when you want raw R6RS Scheme performance without Racket's language framework.
- racket/rhombus — the next-generation "Racket2" language with conventional syntax on the Racket runtime; use when you want Racket's macro power without S-expression surface syntax (still pre-1.0).
- GNU Guile — an embeddable Scheme designed to be scripted into C applications; use when embedding a scripting layer rather than building a standalone app.
- SBCL (Steel Bank Common Lisp) — a high-performance Common Lisp with a mature compiler; use when you want Lisp with strong native performance and CLOS.

## History

| Version | Date | Notes |
|---------|------|-------|
| PLT Scheme | 1995 | Origin as the PLT group's Scheme system and DrScheme IDE. |
| 5.0 | 2010-06 | Renamed from PLT Scheme to Racket[^1]. |
| 6.0 | 2014-05 | Package system reorganization; distribution split into packages. |
| 7.0 | 2018-07 | Racket CS (Chez backend) introduced as an experimental parallel build. |
| 8.0 | 2021-02 | Racket CS becomes the default implementation[^4]. |
| 8.x | 2026 | Active 8-series releases; Rhombus developed in parallel, pre-1.0. |

## References

[^1]: "From PLT Scheme to Racket" — the 2010 renaming announcement. https://racket-lang.org/new-name.html
[^2]: M. Felleisen, R. Findler, M. Flatt, S. Krishnamurthi, E. McCarthy, S. Tobin-Hochstadt, "The Racket Manifesto," SNAPL 2015. https://racket-lang.org/
[^3]: Racket downloads (prebuilt and source distributions). https://download.racket-lang.org
[^4]: Racket blog, "Racket v8.0" release announcement (Racket CS default) — 2021-02. https://blog.racket-lang.org/
[^5]: M. Flatt et al., "Rebuilding Racket on Chez Scheme (Experience Report)," ICFP 2019. https://docs.racket-lang.org/
[^6]: A. Takikawa, D. Feltey, B. Greenman, M. New, J. Vitek, M. Felleisen, "Is Sound Gradual Typing Dead?," POPL 2016. https://doi.org/10.1145/2837614.2837630

## Tags

racket, scheme, lisp, functional-programming, language-oriented-programming, macros, dsl, compiler, gradual-typing, programming-language, education
