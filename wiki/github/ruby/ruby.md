# ruby/ruby

> The reference implementation (CRuby/MRI) of the Ruby programming language — a dynamic, object-oriented, garbage-collected language optimized for programmer happiness.

[GitHub repo](https://github.com/ruby/ruby) ·
[Official website](https://www.ruby-lang.org/) ·
License: Ruby License OR BSD-2-Clause[^1]

## Overview

Ruby is a dynamically typed, object-oriented interpreted language created by Yukihiro Matsumoto ("Matz") in Japan, with the first public release in December 1995[^2]. This repository is the canonical implementation — known as **CRuby** (it is written in C) or **MRI** (Matz's Ruby Interpreter). When people say "Ruby" without qualification, they mean the language as defined by what this repository does; there is an ISO standard (ISO/IEC 30170:2012) but in practice CRuby's behavior is the specification.

Ruby's design philosophy prioritizes developer ergonomics over machine efficiency: everything is an object (including integers and `nil`), blocks and closures are first-class, and the language is aggressively malleable through open classes and metaprogramming. That flexibility is also the defining tension. The same features that make Rails' DSLs and `method_missing` tricks possible make large Ruby codebases hard to statically analyze, and they historically made the interpreter hard to optimize. Ruby's single dominant use case remains web backends via Ruby on Rails; outside that, it competes with Python for scripting and glue work but has a far smaller data/ML ecosystem.

The two structural constraints every serious Ruby operator eventually hits are the **Global VM Lock** (the GVL, sometimes called the GIL — only one thread executes Ruby code at a time, so threads help with I/O concurrency but not CPU parallelism) and the historically slow interpreter. Both are being actively worked on: YJIT (a JIT compiler contributed by Shopify) addresses speed, and Ractor addresses parallelism, though neither is a finished story as of the 3.x line.

## Getting Started

Ruby is best installed via a version manager rather than the system package, so projects can pin versions:

```bash
# macOS/Linux via rbenv (mise and rvm are alternatives)
brew install rbenv
rbenv install 3.4.4
rbenv global 3.4.4
ruby --version
```

A minimal program showing blocks, everything-is-an-object, and duck typing:

```ruby
class Greeter
  def initialize(name)
    @name = name
  end

  def greet = "Hello, #{@name}"
end

people = %w[Tom Brad].map { |n| Greeter.new(n) }
people.each { |g| puts g.greet }

# Enumerable is where most day-to-day Ruby lives:
puts (1..10).select(&:even?).sum   # => 30
```

Dependencies are managed with Bundler (ships with Ruby) and a `Gemfile`:

```bash
bundle init
bundle add sinatra
bundle exec ruby app.rb
```

## Architecture / How It Works

CRuby is a bytecode interpreter, not a tree-walker. The pipeline is:

1. **Parse** — source is tokenized and parsed into an AST. Historically this was `parse.y`, a hand-maintained Bison grammar. Since Ruby 3.3 a new portable parser, **Prism**, has been developed and vendored; it became the default in 3.4[^3]. Prism exists so tools (Sorbet, RuboCop, editors, alternative implementations) can share one parser instead of each reimplementing Ruby's notoriously irregular grammar.
2. **Compile** — the AST is compiled to bytecode for **YARV** (Yet Another Ruby VM), the stack-based virtual machine introduced in Ruby 1.9 by Koichi Sasada[^4]. YARV replaced the 1.8-era AST-walking interpreter and was the last decade's biggest single performance jump.
3. **Execute** — YARV runs the bytecode. When enabled, a JIT compiles hot methods to native code.

**JIT history is messy and worth understanding.** Ruby 2.6 shipped MJIT, which worked by writing C to disk and shelling out to the system C compiler — portable but with awkward warmup and deployment characteristics. Ruby 3.1 introduced **YJIT** (a lazy basic-block-versioning JIT from Shopify), which in Ruby 3.2 was rewritten in Rust and made production-ready[^5]. This is why `rust` appears in this repo's language topics. Ruby 3.3 replaced MJIT with RJIT (a pure-Ruby JIT, mostly a research vehicle); YJIT is the one you actually turn on in production with `--yjit`.

**Memory** is managed by a mark-and-sweep garbage collector that gained generational collection (RGenGC, 2.1) and incremental collection (2.2) to reduce pause times. Objects are 40-byte `RVALUE` slots; larger objects allocate off-heap.

**Concurrency** has three models: OS threads (subject to the GVL), Fibers (cooperative, with a pluggable Fiber Scheduler since 3.0 enabling async I/O libraries like the `async` gem), and **Ractor** (3.0) — an actor model that runs Ruby code in genuine parallel by requiring objects passed between Ractors to be shareable/frozen. Ractor is still marked experimental years later; most real parallelism in production is still achieved by forking processes (Unicorn, Puma workers, Sidekiq).

## Production Notes

**The GVL shapes every deployment decision.** Because one process runs Ruby on one core at a time, CPU-bound web apps scale by running multiple processes, not multiple threads. Puma uses a hybrid (workers × threads) model where threads only help while other threads are blocked on I/O. Sizing worker count to cores and watching memory-per-worker is the core operational skill.

**Memory bloat and forking.** Long-running Ruby processes tend to grow. Copy-on-write friendliness (so forked workers share pages) depends on GC behavior; `jemalloc` is a common swap-in to reduce fragmentation, and tools like `puma_worker_killer` restart bloated workers. Set `MALLOC_ARENA_MAX` on glibc to avoid runaway arena allocation.

**YJIT is opt-in and has a memory/speed tradeoff.** It is off by default; enable with `RUBY_YJIT_ENABLE=1` or `--yjit`. It meaningfully speeds up Rails-shaped workloads but adds executable memory overhead, tunable via `--yjit-exec-mem-size`. Benchmark your own app — gains vary widely by workload and are smaller for I/O-bound endpoints.

**Upgrade pains.** The 1.8 → 1.9 jump (string encodings, YARV, ordered hashes) was the most disruptive in Ruby's history and split the ecosystem for years. Since 2.0, upgrades are far smoother, but watch for: the frozen-string-literal migration (a magic comment today, planned default later — code that mutates string literals breaks), keyword-argument separation finalized in 3.0 (a real source of breakage from 2.7 deprecation warnings), and the parser switch to Prism in 3.4 which can surface edge-case syntax differences.

**Native extension gems.** Gems with C extensions (`nokogiri`, `pg`, `mysql2`, `grpc`) compile at install time and are the usual cause of `bundle install` failures in CI and Docker — they need matching headers and a toolchain. Precompiled platform gems have made `nokogiri` painless, but the general failure mode remains.

**Where the bug tracker actually is.** Feature and bug discussion happens on the Redmine at bugs.ruby-lang.org, not GitHub Issues; GitHub is used for pull requests. The open-issue count here reflects PRs, not the language's real backlog.

## When to Use / When Not

**Use when:**
- You're building a web backend and want Rails' productivity — this is Ruby's strongest, best-supported path.
- You want highly readable code and fast iteration for CRUD-heavy business applications.
- You're writing scripts, build tooling, or DSL-heavy configuration (Fastlane, Chef/Puppet, Homebrew are all Ruby).
- Team velocity and maintainability matter more than raw single-thread throughput.

**Avoid when:**
- You need CPU-bound parallelism in one process — the GVL makes threads unhelpful for that, and Ractor is not yet production-mature.
- You're doing data science / ML — Python's ecosystem is overwhelmingly larger.
- You need predictable low-latency or minimal memory footprint (systems programming, embedded) — reach for Go, Rust, or mruby.
- Startup time or single-binary distribution is critical — the interpreter and gem loading add overhead.

## Alternatives

- oracle/truffleruby — GraalVM-based Ruby with a strong JIT; can be much faster on CPU-bound code but has heavier startup and occasional compatibility gaps. Use when peak throughput matters and you can tolerate a non-MRI runtime.
- jruby/jruby — Ruby on the JVM with real thread parallelism (no GVL) and Java interop. Use when you need JVM integration or genuine multithreading and can accept slower startup.
- mruby/mruby — a lightweight, embeddable Ruby for constrained/embedded environments. Use when you need Ruby scripting inside a C/C++ host or on tiny targets.
- crystal-lang/crystal — a compiled, statically typed language with Ruby-like syntax. Use when you want Ruby's feel with native performance and type checking, accepting a much smaller ecosystem and no true source compatibility.
- python/cpython — the closest peer for general scripting/glue and the default for data/ML work. Use instead when the ecosystem (not the language) is what you need.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.95 | 1995-12 | First public release by Matz[^2]. |
| 1.8 | 2003-08 | Long-lived stable line; large stdlib growth. |
| 1.9 | 2007–08 | YARV bytecode VM, string encodings, ordered hashes — the big break[^4]. |
| 2.0 | 2013-02 | Refinements, keyword arguments, lazy enumerators. |
| 2.1 | 2013-12 | Generational GC (RGenGC). |
| 2.3 | 2015-12 | Frozen string literal pragma, safe navigation `&.`. |
| 2.6 | 2018-12 | First JIT (MJIT). |
| 3.0 | 2020-12 | "Ruby 3x3" target; Ractor, Fiber Scheduler, RBS/TypeProf, keyword-arg separation[^6]. |
| 3.1 | 2021-12 | YJIT introduced (from Shopify). |
| 3.2 | 2022-12 | YJIT rewritten in Rust, production-ready; WASI/WASM support[^5]. |
| 3.3 | 2023-12 | Prism parser added, RJIT replaces MJIT, M:N thread scheduler. |
| 3.4 | 2024-12 | Prism becomes the default parser; `it` implicit block parameter[^3]. |

## References

[^1]: Ruby is distributed under a dual license — the Ruby License or the 2-clause BSD License. GitHub's license detector reports "NOASSERTION" because the terms are custom rather than a standard SPDX identifier. See the COPYING file. https://github.com/ruby/ruby/blob/master/COPYING
[^2]: "About Ruby" and Ruby history — ruby-lang.org. https://www.ruby-lang.org/en/about/
[^3]: Prism parser project. https://github.com/ruby/prism
[^4]: YARV (Yet Another Ruby VM), Koichi Sasada. https://en.wikipedia.org/wiki/YARV
[^5]: "YJIT: Building a New JIT Compiler for CRuby" — Shopify Engineering. https://shopify.engineering/yjit-just-in-time-compiler-cruby
[^6]: "Ruby 3.0.0 Released" — ruby-lang.org, 2020-12-25. https://www.ruby-lang.org/en/news/2020/12/25/ruby-3-0-0-released/

## Tags

ruby, programming-language, interpreter, cruby, mri, yjit, dynamic-typing, object-oriented, garbage-collection, web-backend, c, jit
