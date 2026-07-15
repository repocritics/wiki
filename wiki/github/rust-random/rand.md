# rust-random/rand

> Rust's de facto random number generation library — a family of crates over a shared `RngCore` trait, covering generators, distributions, and sequence sampling.

[GitHub repo](https://github.com/rust-random/rand) ·
[crates.io](https://crates.io/crates/rand) ·
[License: MIT OR Apache-2.0](https://github.com/rust-random/rand/blob/master/COPYRIGHT)

## Overview

`rand` is the standard-in-practice random number library for Rust. The Rust standard library deliberately ships no RNG, so almost any program that needs randomness — games, simulations, test data, token generation, shuffling — pulls in `rand` or one of its sibling crates. It is maintained by the community-run `rust-random` organization, not the Rust project itself, though it originated inside `rust-lang` before being split out[^1].

The design is a layered family of crates rather than one monolith. At the bottom is `rand_core`, which defines the `RngCore` and `SeedableRng` traits that every generator implements. On top of that sit concrete generators (`StdRng`, `SmallRng`, `ThreadRng`), the distributions layer (`StandardUniform`, `Uniform`, and the larger `rand_distr` crate), and sequence operations (`choose`, `shuffle`) in `rand::seq`. This layering is the library's central design decision: it lets a numeric crate depend only on `rand_core` traits without pulling in the full generator and distribution machinery, while application code gets the batteries-included `rand` facade[^2].

The defining tension is **correctness and flexibility over simplicity**. The maintainers state plainly that `rand` is neither small nor simple, and point users who want a tiny dependency to `fastrand` or `oorandom` instead[^2]. The flip side is a real cost: the crate has a history of breaking API churn across 0.x releases (it is still pre-1.0 as of 2026), and the split-crate structure means a single logical upgrade can touch `rand`, `rand_core`, `rand_distr`, and `getrandom` version constraints at once.

## Getting Started

```bash
cargo add rand
```

```rust
use rand::Rng;

fn main() {
    // Automatically-seeded thread-local generator.
    let mut rng = rand::rng();

    let n: u32 = rng.random();                 // any u32
    let dice = rng.random_range(1..=6);        // inclusive range
    let coin: bool = rng.random_bool(0.5);     // Bernoulli

    let mut deck: Vec<u32> = (1..=52).collect();
    use rand::seq::SliceRandom;
    deck.shuffle(&mut rng);

    println!("{n} {dice} {coin} {:?}", &deck[..3]);
}
```

Note the 0.9+ API names (`rng()`, `random()`, `random_range()`). Older tutorials use `thread_rng()`, `gen()`, and `gen_range()`, which were renamed — see Production Notes.

## Architecture / How It Works

**The `RngCore` trait** is the contract every generator implements: `next_u32`, `next_u64`, and `fill_bytes`. It lives in `rand_core` so that generator crates and consumer crates can depend on the trait without depending on `rand` proper. `SeedableRng` adds deterministic construction from a seed and from another RNG.

**The `Rng` extension trait** (in `rand`) is where ergonomic methods live: `random()`, `random_range()`, `random_bool()`, sampling from distributions. It is a blanket impl over any `RngCore`, so the split between "implement the core" and "use the sugar" is clean.

**Generators shipped by `rand`:**
- `StdRng` — a cryptographically-motivated PRNG. Since 0.6 it has been backed by ChaCha (a ChaCha-family block cipher), currently via the `chacha20` crate. It is a portable, reproducible generator when explicitly seeded.
- `SmallRng` — a fast, small, non-cryptographic PRNG for simulations and games where speed matters and unpredictability does not. The concrete algorithm is not guaranteed stable across releases.
- `ThreadRng` / `rng()` — a lazily-seeded, thread-local, automatically-reseeding generator. It is the default for casual use and is reasonably strong but is not a stability or cryptographic guarantee.

**Seeding.** Entropy from the OS comes through the `getrandom` crate, which abstracts over platform CSPRNGs (`getrandom(2)`, `/dev/urandom`, `BCryptGenRandom`, etc.). `getrandom` is a frequent source of build/target friction — see below.

**Distributions.** `StandardUniform` samples a "natural" uniform value for a type; `Uniform` samples within a range with rejection sampling to avoid modulo bias. Non-uniform distributions (Normal, Poisson, Gamma, Exponential, Weighted sampling, etc.) live in the separate `rand_distr` crate to keep `rand`'s dependency and compile surface smaller.

**Other RNG crates** in the ecosystem plug into the same traits: `rand_pcg` (PCG family), `rand_xoshiro` (xoshiro/xoroshiro), `rand_chacha`/`chacha20`, `rand_seeder`. Because they only implement `RngCore`/`SeedableRng`, they are drop-in interchangeable.

## Production Notes

**Reproducibility is a narrow, explicit contract.** `rand` guarantees value-stable output only for a documented subset — a specific seeded generator plus specific distributions[^3]. `ThreadRng`, `SmallRng`, and anything relying on default-seeding are explicitly *not* reproducible across versions or platforms. For simulations that must replay identically, seed a named portable generator (e.g. a `rand_pcg` or `rand_chacha` type) directly and pin its version; do not rely on `StdRng` staying byte-identical across a major bump.

**The 0.9 rename is the biggest recent upgrade footgun.** Across the 0.8 → 0.9 transition, `thread_rng()` became `rng()`, `gen()` became `random()`, `gen_range()` became `random_range()`, the `distributions` module became `distr`, and `Standard` became `StandardUniform`. Code and answers written before 2025 use the old names, so mixing tutorial snippets with a current `rand` produces confusing "method not found" errors. Read the crate's own upgrade guide rather than trusting search results[^4].

**`getrandom` and WebAssembly.** The `wasm32-unknown-unknown` target has no OS entropy source, so it is not supported automatically — you must configure a `getrandom` backend (or disable the OS-RNG feature) or the build fails at link/run time[^5]. WASI and Emscripten targets are supported directly. This bites teams shipping browser WASM the first time they cross-compile.

**Version-constraint coupling.** Because `rand`, `rand_core`, `rand_distr`, and `getrandom` are separately versioned, mismatched constraints in a dependency tree can force multiple copies of `getrandom` or fail to unify `RngCore`. When upgrading, bump the whole family together and check `cargo tree -d` for duplicates.

**Cryptographic use requires care.** `rand` is not primarily a crypto library. Some generators aim to be unpredictable, but the maintainers push the decision of "is this good enough for my threat model" onto the user and maintain a separate `SECURITY.md`[^6]. For key material and nonces, prefer purpose-built crates (`getrandom` directly, or a vetted CSPRNG) and read that document first.

**`no_std`.** Supported but partial: `alloc` gates a large fraction of sequence and distribution functionality, and `std`-only pieces (like OS seeding and `ThreadRng`) drop out. Embedded users typically pair `rand_core` with an explicitly-seeded PRNG.

## When to Use / When Not

**Use when:**
- You need general-purpose randomness in Rust and want the ecosystem-standard, well-reviewed option.
- You need a breadth of distributions or weighted/sequence sampling (`rand` + `rand_distr`).
- You want interchangeable generators behind one trait (swap `SmallRng` for a seeded PCG without touching call sites).

**Avoid (or reach for something else) when:**
- You want a tiny, single-file dependency with a stable trivial API — `fastrand` or `oorandom` are simpler.
- You need cryptographic key/nonce generation specifically — go to `getrandom` or a dedicated crypto crate, not `rand`'s conveniences.
- You need byte-for-byte reproducibility across versions without pinning — the default generators do not promise it.

## Alternatives

- smol-rs/fastrand — minimal, fast, non-cryptographic PRNG in one small crate; use when you want randomness with almost no dependency weight and don't need distributions.
- Lokathor/randomize / `oorandom` — tiny PCG-only generators; use when you want a single deterministic algorithm and nothing else.
- rust-random/getrandom — OS entropy only; use directly when you just need cryptographically-secure random bytes and no distribution/sequence sugar.
- statrs-dev/statrs — statistical library with its own distributions; use when statistics (not just sampling) is the primary need.
- dhardy/rand_distr (part of this org) — the distributions companion; use alongside `rand` when you need Normal/Poisson/Gamma/weighted sampling.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.4 | 2017 | Pre-split era; `rand` still a single crate. |
| 0.5 | 2018 | Major redesign; `rand_core` trait split introduced. |
| 0.6 | 2018 | `StdRng` moved to a ChaCha-based generator. |
| 0.7 | 2019 | OS seeding routed through the `getrandom` crate; `SmallRng` reworked. |
| 0.8 | 2020-12 | API cleanups; `Rng` trait refinements. |
| 0.9 | 2025 | Large rename: `rng()`, `random()`, `random_range()`, `distr`, `StandardUniform`[^4]. |
| 0.10 | 2026-02 | Latest `MAJOR.MINOR` line; still pre-1.0[^2]. |

## References

[^1]: rust-random organization and the Rust Rand Book. https://rust-random.github.io/book/
[^2]: `rand` README and crate overview ("Rand is mature but not yet at 1.0"; "not small, not simple"). https://github.com/rust-random/rand
[^3]: The Rust Rand Book — Portability / reproducibility. https://rust-random.github.io/book/portability.html
[^4]: The Rust Rand Book — Upgrade Guide. https://rust-random.github.io/book/update.html
[^5]: `getrandom` WebAssembly support notes, referenced from the `rand` README. https://docs.rs/getrandom/latest/getrandom/#webassembly-support
[^6]: `rand` SECURITY.md. https://github.com/rust-random/rand/blob/master/SECURITY.md

## Tags

rust, random, rng, prng, csprng, random-number-generation, distributions, sampling, cargo-crate, no-std
