# grame-cncm/faust

> A functional language for real-time audio DSP that compiles a block-diagram algebra down to efficient C++, LLVM IR, WebAssembly, Rust, and a dozen other backends.

[GitHub repo](https://github.com/grame-cncm/faust) ·
[Official website](https://faust.grame.fr) ·
License: GPL with FAUST exception (GitHub reports NOASSERTION)

## Overview

Faust (Functional Audio Stream) is a domain-specific language for signal processing and sound synthesis, developed since the early 2000s by Yann Orlarey and colleagues at GRAME, Centre National de Creation Musicale in Lyon[^1]. It is not a library or a runtime — it is a *compiler*. You write a DSP algorithm as a functional block-diagram expression, and Faust emits source code (C, C++, LLVM IR, WebAssembly, Rust, Dlang, Julia, JAX, C#, Cmajor, JSFX, and more) that operates at sample level[^2]. That generated code is then wrapped by an "architecture file" to become a standalone app, an iOS/Android app, or a plugin for VST, AU, LADSPA, LV2, Max/MSP, Pure Data, SuperCollider, Csound, and others.

The defining property is that Faust is *fully compiled and statically structured*. A `.dsp` program describes a fixed signal graph; there is no dynamic patching at runtime the way there is in Pure Data or Max. This buys very high performance (the compiler does aggressive normalization, common-subexpression elimination, and vectorization) and portability across an unusually large target matrix — at the cost of a steep, unfamiliar mental model. Faust is two languages stacked: a compile-time "box" metalanguage that builds the diagram, and the "signal" semantics the diagram denotes. Newcomers routinely conflate the two.

The audience is DSP researchers, plugin developers, computer-music practitioners, and educators. Faust is widely taught (Stanford CCRMA, GRAME workshops) and is the code-generation engine behind zero-install web tools like the Faust IDE and Faust Playground, which compile to WebAssembly in the browser[^3].

## Getting Started

```bash
# macOS
brew install faust
# Debian/Ubuntu: apt-get install faust  (or build from source via CMake since 2.5.18)
```

A minimal program — a 440 Hz sine at half gain, with a slider for frequency:

```faust
import("stdfaust.lib");

freq = hslider("freq", 440, 20, 2000, 0.1);
process = os.osc(freq) * 0.5;
```

Compile it. `faust` alone prints C++ to stdout; the `faust2...` wrapper scripts drive a full native toolchain:

```bash
faust noise.dsp                 # emit C++ to stdout
faust -lang rust noise.dsp      # emit Rust instead
faust2jaqt synth.dsp            # build a JACK + Qt standalone app
faust2lv2 synth.dsp             # build an LV2 plugin
```

The five composition operators are the core of the language: `:` (sequential), `,` (parallel), `<:` (split/fan-out), `:>` (merge/fan-in), and `~` (recursion, for feedback and filters).

## Architecture / How It Works

The pipeline runs front-end to back-end:

1. **Parse** — a `.dsp` file is parsed into the *box language*, a term algebra of block diagrams. The box layer is evaluated at compile time; it is where `import`, pattern matching, `par`/`seq` iterations, and metaprogramming live.
2. **Signal generation** — boxes are lowered into a *signal graph*: functions from input signals to output signals, one value per sample. Rate inference, type inference (int/float, and constant vs. sample-varying), and delay-line sizing happen here.
3. **Symbolic normalization** — the signal graph is simplified: constant folding, common-subexpression elimination, and delay-line sharing. This pass is where Faust's performance comes from and is largely backend-independent.
4. **Code generation** — a pluggable backend walks the normalized graph and emits target code. Backends include C, C++, LLVM IR, WebAssembly, Rust, Dlang, Julia, JAX, C#, Cmajor, and JSFX[^2].

Two artifacts sit around the generated DSP class. **Architecture files** (`architecture/`) adapt the pure DSP class to a host: they provide audio I/O, GUI widget binding, and the plugin ABI (VST3, AU, LV2, PD external, etc.). The **`faust2...` scripts** (`tools/`) chain the compiler, the chosen architecture file, and the platform's native compiler into one command.

**libfaust** embeds the whole compiler as a C++ library. Combined with the LLVM backend and LLVM's JIT, it lets host programs compile Faust code *at runtime* — this is how FaustLive, FaucK (Faust-in-ChucK), and `faustgen~` (Faust-in-Max) work. Embedding therefore pulls in LLVM as a heavyweight dependency.

The DSP **libraries** (`stdfaust.lib` and friends) were split into a [separate repository](https://github.com/grame-cncm/faustlibraries) and are pulled in as a git submodule[^4]. The repository's default branch is `master-dev`, not `master`; `master` is kept release-stable and merged only at release time[^5].

## Production Notes

**Static structure is the central tradeoff.** Because the graph is fixed at compile time, you cannot add or remove voices, filters, or routing at runtime the way a Pd patch can. Polyphony and dynamic voice allocation are handled by architecture-level `dsp_poly` wrappers around a fixed voice DSP, not by the language. Any structural change means recompiling.

**Sample-at-a-time is the natural grain.** Faust denotes per-sample signal functions, so block/frame algorithms — FFT, partitioned convolution, spectral processing — are awkward to express and were historically weak. Library support exists but block processing is not where the language is comfortable. If your algorithm is inherently frame-based, expect friction.

**Delay lines are bounded at compile time.** The `@` delay and table sizes must be statically known (or bounded), because buffer sizes are allocated at code-gen. Unbounded/variable-length delays require care.

**Compiler diagnostics are terse.** Connection-count mismatches (an expression producing the wrong number of output signals for the next block's inputs) and box-vs-signal confusion produce error messages that are hard to read until you internalize the algebra. This is the most common beginner wall.

**Licensing needs a lawyer's eye, not a glance.** GitHub reports the repo license as `NOASSERTION`. The Faust *compiler* is GPL, but GRAME grants a well-known exception so that code generated by the compiler is *not* forced under the GPL — commercial closed-source plugins built from Faust output are explicitly permitted[^6]. However, individual architecture files and the DSP libraries carry their own licenses (variously GPL-with-exception, LGPL-style, or STK-derived terms). If you ship a product, verify the license of every architecture file and library your build actually links.

**WebAssembly is a first-class, well-exercised path.** The web IDE/Playground and the `faustwasm` toolchain compile to WASM and run in-browser. SIMD and thread support depend on the runtime; measure before assuming.

## When to Use / When Not

**Use when:**
- You want one DSP source to target many plugin formats and platforms without rewriting for each SDK.
- You need high-performance, low-level sample processing and want the compiler to vectorize and simplify for you.
- You're doing DSP research or teaching and value concise, mathematically-grounded block-diagram semantics.
- You want in-browser or embeddable (libfaust/JIT) audio compilation.

**Avoid when:**
- Your app needs a user-editable, runtime-reconfigurable signal graph — use Pure Data or Max/Gen instead.
- Your algorithm is fundamentally frame/spectral-based (heavy FFT, block convolution) rather than per-sample.
- The team wants C-family syntax and is unwilling to invest in the functional block algebra and its two-layer mental model.
- You need a full application framework (GUI, state, hosting) — Faust generates the DSP core, not the app around it.

## Alternatives

- pure-data/pure-data — visual dataflow, interpreted and runtime-patchable; use when end users need to rewire the graph live rather than ship a compiled binary.
- supercollider/supercollider — server + `sclang` synthesis environment; use when you want live-coding and pattern/scheduling infrastructure, not a codegen compiler.
- csound/csound — mature audio DSP language with decades of opcodes and score-based composition; use when you need that breadth and legacy ecosystem.
- cmajor-lang/cmajor — C-like compiled DSP language targeting native and WASM; use when you prefer C-family syntax over Faust's functional algebra (Faust can even emit Cmajor).
- juce-framework/JUCE — C++ plugin/app framework, not a DSP language; use when you hand-write DSP in C++ and need the hosting/GUI scaffolding that Faust's architecture files otherwise provide.

## History

| Version | Date | Notes |
|---------|------|-------|
| Origin | early 2000s | Created at GRAME by Yann Orlarey; functional block-diagram DSP algebra[^1]. |
| — | 2009 | Foundational paper formalizing Faust's functional approach to DSP[^1]. |
| 0.9.x | pre-2017 | Original single-target (C++) compiler. |
| 2.0 | c. 2017 | Rewrite introducing multiple backends (C/C++/LLVM/WASM) and libfaust JIT embedding[^2]. |
| 2.5.18 | — | Build and install moved to CMake[^7]. |
| 2.x | 2020–2024 | Rust, Julia, JAX, Cmajor, C#, and JSFX backends added; WebAssembly tooling matured[^2]. |

## References

[^1]: Y. Orlarey, D. Fober, S. Letz, "FAUST: an Efficient Functional Approach to DSP Programming" (2009), GRAME. https://faust.grame.fr
[^2]: Faust manual — supported output languages and backends. https://faustdoc.grame.fr/manual/overview/
[^3]: Faust IDE and Faust Playground — in-browser WebAssembly compilation. https://faustide.grame.fr
[^4]: faustlibraries — DSP libraries as a separate repository / submodule. https://github.com/grame-cncm/faustlibraries
[^5]: grame-cncm/faust README — `master` vs `master-dev` branch policy. https://github.com/grame-cncm/faust
[^6]: Faust licensing and the compiler-output exception. https://github.com/grame-cncm/faust
[^7]: grame-cncm/faust README — CMake-based build since release 2.5.18. https://github.com/grame-cncm/faust

## Tags

audio, dsp, sound-synthesis, functional-programming, compiler, programming-language, cpp, webassembly, audio-plugins, real-time, code-generation
