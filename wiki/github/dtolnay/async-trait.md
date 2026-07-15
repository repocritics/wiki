# dtolnay/async-trait

> Attribute macro that makes `async fn` in traits work with `dyn Trait` by rewriting each method to return a boxed future.

[GitHub repo](https://github.com/dtolnay/async-trait) ·
[docs.rs](https://docs.rs/async-trait) ·
[License: Apache-2.0 OR MIT](https://github.com/dtolnay/async-trait/blob/master/LICENSE-APACHE)

## Overview

`async-trait` is a procedural macro that solves one specific gap in the Rust
language: `async fn` in traits that stay usable behind `dyn`. Native `async fn`
in traits (AFIT) stabilized in Rust 1.75 (December 2023)[^1], but a trait with a
native `async fn` is not dyn-compatible (previously "object-safe"), so
`Box<dyn Trait>` and `&dyn Trait` do not compile. The macro sidesteps this by
rewriting every `async fn` into an ordinary `fn` that returns
`Pin<Box<dyn Future + Send + 'async_trait>>`, which *is* dyn-compatible[^2].

It is one of dtolnay's (David Tolnay) many single-purpose proc-macro crates —
the same author behind serde, syn, quote, anyhow, and thiserror. The design
shows: no runtime dependency, no `unsafe` in the generated code, and a narrow
contract. As of 2026 it remains one of the most downloaded crates on crates.io
despite the language having partially subsumed its use case.

The defining tension is that `async-trait` is a stopgap the language is slowly
retiring. For any trait you own and always use statically, native AFIT is now
the better choice — no macro, no allocation. `async-trait` earns its place only
where you need dynamic dispatch (`dyn`) over async methods, which the language
still does not provide natively[^2]. Treat it as targeted, not default.

## Getting Started

```bash
cargo add async-trait
```

```rust
use async_trait::async_trait;

#[async_trait]
trait Fetcher {
    async fn get(&self, url: &str) -> Result<String, std::io::Error>;
}

struct Http;

#[async_trait]
impl Fetcher for Http {
    async fn get(&self, url: &str) -> Result<String, std::io::Error> {
        // await other futures freely inside the body
        Ok(format!("fetched {url}"))
    }
}

// The payoff: this trait is now dyn-compatible.
async fn run(f: &dyn Fetcher) {
    let _ = f.get("https://example.com").await;
}
```

The macro must be applied to **both** the trait definition and every `impl`
block. Forgetting one side produces a mismatch error, not silent misbehavior.

## Architecture / How It Works

The transformation is mechanical and fully visible in the expanded output. An
`async fn f(&self) -> T` becomes:

```rust
fn f<'async_trait>(&'async_trait self)
    -> Pin<Box<dyn Future<Output = T> + Send + 'async_trait>>
where Self: Sync + 'async_trait
{
    Box::pin(async move { /* original body */ })
}
```

Three things are happening. First, the async body is wrapped in an `async move`
block and heap-allocated via `Box::pin` — a future of unknown size becomes a
fixed-size pointer, which is what makes the method dyn-compatible. Second, an
`'async_trait` lifetime is synthesized to tie the returned future to the borrows
it captures (`&self`, reference arguments). Third, a `Send` bound is added to
the boxed future by default so the trait object can cross threads, which is what
multithreaded executors like Tokio require.

The `Send` default is the most common friction point. If your futures legitimately
cannot be `Send` (they hold an `Rc`, a raw pointer, a non-`Send` guard across an
`.await`), apply `#[async_trait(?Send)]` to both the trait and its impls to drop
the bound. This is all-or-nothing per trait, not per method.

Lifetime elision is the other sharp edge. Rust's `async fn` desugaring does not
allow implicit elided lifetimes outside `&`/`&mut`, so a parameter like
`Elided` (a type alias for `&'a T`) must be written `Elided<'_>` or named
explicitly[^3]. The compiler diagnoses this clearly, but it surprises people who
never wrote the lifetime before adding the macro.

There is no `unsafe` in the generated code and no runtime component — the crate
is purely a compile-time rewrite. If it compiles, the soundness guarantees are
the same as hand-written boxed-future code.

## Production Notes

**Every call allocates.** `Box::pin` heap-allocates one future per method
invocation. For most I/O-bound code this is noise next to the syscall, but in
hot loops or latency-sensitive inner paths the allocation and the dynamic
dispatch through the vtable are measurable. If a trait is only ever used
statically, native AFIT avoids both costs entirely.

**The `Send` bound propagates.** Because the default rewrite adds `+ Send`, every
future you `.await` inside a method body must itself be `Send`, and any captured
state held across an `.await` must be `Send`. A non-`Send` type deep in a call
chain surfaces as a confusing error at the trait boundary. `?Send` fixes it but
then the trait object cannot be shared across threads — a real tradeoff, not a
free switch.

**Apply-to-both-sides is a maintenance tax.** The macro must decorate the trait
and every impl. In large codebases this is easy to forget when adding a new impl,
and the resulting error points at the impl, not the omission. Some teams wrap the
attribute or lint for it.

**Interaction with native AFIT.** Since Rust 1.75 you can mix native `async fn`
traits and `async-trait` traits in one codebase. Do not put `#[async_trait]` on a
trait that does not need `dyn` — you pay the allocation for nothing. The
`trait-variant` crate[^4] offers a lower-overhead path for the Send-bound problem
on native AFIT (a `Send`-bounded variant without boxing) and is the recommended
direction when you control the trait and don't need trait objects.

**Error-message quality.** Because the macro rewrites signatures, type errors
sometimes reference the synthesized `'async_trait` lifetime or the boxed future
rather than your source. Reading `cargo expand` output is the fastest way to
debug a stubborn case.

## When to Use / When Not

**Use when:**
- You need `dyn Trait` (trait objects) over async methods — plugin registries,
  `Vec<Box<dyn Handler>>`, heterogeneous collections of async implementers.
- You must support a wide range of compiler versions, including pre-1.75.
- You want async trait methods with full generics/lifetimes/associated types and
  don't want to hand-roll boxed futures.

**Avoid when:**
- The trait is always used statically (generic bounds, `impl Trait`) — native
  `async fn` in traits is allocation-free and needs no macro.
- You are in an allocation-sensitive hot path and can restructure to avoid `dyn`.
- You only need a `Send`-bounded native trait — reach for `trait-variant` instead
  of paying for boxing.

## Alternatives

- rust-lang/rust — native `async fn` in traits (stable since 1.75) covers the
  static-dispatch case with zero allocation; use it when you don't need `dyn`.
- rust-lang/impl-trait-utils (trait-variant) — generates a `Send`-bounded variant
  of a native async trait; use it when you control the trait and want the Send
  ergonomics of `async-trait` without boxing.
- dtolnay/async-recursion — sibling macro specifically for recursive async fns
  that need boxing; use it when the problem is recursion, not trait objects.
- Manual `Pin<Box<dyn Future>>` methods — write the desugaring yourself when you
  want zero macro dependency and full control over bounds.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019-07 | Initial release; boxed-future rewrite for async trait methods[^5]. |
| — | 2019-11 | `?Send` option added for non-threadsafe futures. |
| — | 2023-12 | Rust 1.75 stabilizes native AFIT; `async-trait` scope narrows to `dyn` use[^1]. |
| 0.1.x | 2026-06 | Still actively maintained; 0.1 series, no breaking 1.0 planned. |

The crate has deliberately stayed on the 0.1 line for its entire life — a signal
of API stability, not immaturity. Last commit June 2026; ~2.2k stars and heavy
crates.io download volume make it de facto infrastructure for the async Rust
ecosystem.

## References

[^1]: Rust Blog, "Announcing Rust 1.75.0" — `async fn` and return-position
`impl Trait` in traits stabilized, 2023-12-28.
https://blog.rust-lang.org/2023/12/28/Rust-1.75.0.html
[^2]: async-trait README — the stabilization of AFIT in 1.75 did not include
`dyn Trait` support; the macro provides it.
https://github.com/dtolnay/async-trait
[^3]: async-trait README, "Elided lifetimes" — async fn syntax forbids implicit
elided lifetimes outside `&`/`&mut`. https://github.com/dtolnay/async-trait
[^4]: trait-variant (rust-lang/impl-trait-utils) — Send-bounded variants for
native async traits. https://github.com/rust-lang/impl-trait-utils
[^5]: crates.io — async-trait release history. https://crates.io/crates/async-trait/versions

## Tags

rust, proc-macro, async, futures, trait-objects, dyn-compatibility, tokio, concurrency, boxed-future, dtolnay
