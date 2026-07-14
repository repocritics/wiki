# tokio-rs/tracing

> Structured, application-level diagnostics for Rust — spans and events with a pluggable subscriber backend, decoupled from any one logger.

[GitHub repo](https://github.com/tokio-rs/tracing) ·
[Official website](https://tracing.rs) ·
[License: MIT](https://github.com/tokio-rs/tracing/blob/main/LICENSE)

## Overview

`tracing` is the de facto logging and diagnostics facade for the Rust ecosystem. It was built by the Tokio project to solve a problem that line-based logging cannot: in concurrent and async code, a single logical unit of work (a request, a task) interleaves with thousands of others, so a flat stream of log lines loses the causal structure. `tracing` models that structure directly with two primitives — **spans** (a period of time, with a beginning and end) and **events** (a moment within, or outside, a span)[^1]. Both carry typed, structured key-value fields rather than only pre-formatted strings.

It is a facade, not a logger. The `tracing` crate is what libraries and applications call (`info!`, `span!`, `#[instrument]`); it does almost nothing on its own. What happens to the collected data — formatting, filtering, sampling, export to a file or to OpenTelemetry — is decided by a `Subscriber` implementation the application installs, most commonly from the companion `tracing-subscriber` crate. This split is the defining design decision: libraries instrument once, applications choose the backend, and neither is coupled to the other[^1]. Though maintained by the Tokio project, `tracing` does not require the Tokio runtime and is widely used in synchronous code.

The defining tension is complexity for correctness. `tracing` is materially harder to adopt than the older `log` crate — the span/subscriber/layer model has real surface area, and async span management has a well-known footgun (see below). In return it gives you context propagation across `.await` points, structured fields, per-target level filtering, and a single instrumentation surface that can feed logs, metrics, and distributed traces at once. For most non-trivial async Rust services it is the standard choice.

## Getting Started

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = "0.3"
```

```rust
use tracing::{info, instrument, Level};

#[instrument]
fn process(order_id: u64) {
    // Fields are structured, not just interpolated into the message.
    info!(items = 3, "processing order");
}

fn main() {
    // Install a global subscriber; RUST_LOG controls the filter.
    tracing_subscriber::fmt()
        .with_max_level(Level::INFO)
        .init();

    process(42); // span "process{order_id=42}" wraps the event
}
```

Libraries should depend only on `tracing` and must **not** install a global subscriber; that is the application's job. The `#[instrument]` attribute creates and enters a span for the duration of the function call, recording its arguments as fields.

## Architecture / How It Works

The ecosystem is layered into distinct crates with different stability guarantees:

- **`tracing-core`** — the minimal, stable API surface: the `Subscriber` trait, `Metadata`, `Callsite`, `Dispatch`, and the `Id` type. Subscriber authors depend on this because it changes rarely. It has no dependency on `std` features it can avoid, enabling `no_std` usage.
- **`tracing`** — the instrumentation front end: the `info!`/`warn!`/`span!` macros, `Span`, and the `#[instrument]` proc-macro (via `tracing-attributes`). Callsites are registered statically, so a disabled span/event is close to a single atomic load and branch — the cost of instrumentation you are not collecting is meant to be near zero.
- **`tracing-subscriber`** — where most operational logic lives: the `Registry` (a `Subscriber` that stores span data), the `Layer` trait, `EnvFilter`, and the `fmt` layer. `Layer`s compose: a `Registry` provides span storage and each `Layer` (formatting, filtering, OpenTelemetry export) stacks on top and sees the same events. This composition model is the real architecture; you build a subscriber by stacking layers.

Data flow: a macro invocation checks its static callsite's cached interest, and if enabled, constructs an event/span and hands it to the current `Dispatch` (thread-local or global). The subscriber receives `new_span`, `enter`, `exit`, `event`, and `close` callbacks. Span storage and parent/child relationships are the subscriber's responsibility — `tracing-core` only assigns IDs.

The most important internal concept for users is span **entry** vs **lifetime**. Entering a span (`span.enter()` returns a guard) marks it as current on this thread until the guard drops. In synchronous code this maps cleanly to a lexical scope. In async code it does not: holding an entered-span guard across an `.await` keeps the span entered even while the task is parked and another task runs, producing incorrect, tangled context[^2]. The correct pattern is `Future::instrument` (or `#[instrument]` on the `async fn`), which enters the span only while the future is being polled. This is the single largest source of `tracing` bugs in the wild.

## Production Notes

**The async span footgun.** Never hold a `span.enter()` guard across `.await`. Clippy's `tracing` lints and the `#[instrument]` macro exist largely to steer you away from it. If your traces show spans that "leak" into unrelated work, this is almost always the cause[^2].

**Filtering is deceptively subtle.** There are two filter mechanisms — global `EnvFilter` (interest-based, evaluated once per callsite, cheap) and per-layer filters (`Filter` trait, evaluated per event). They interact: a per-layer filter that is more permissive than the global interest can silently see nothing, because the global filter already told the callsite it was disabled. `with_filter` on a layer vs. a global `EnvFilter` produce different behavior, and mixing them is a common source of "my logs disappeared" reports.

**`log` interop is one-directional and lossy.** `tracing-log` bridges `log`-crate records into `tracing` events, which is essential because much of the crate ecosystem still emits `log`. But bridged records arrive without the structured fields or span context that native `tracing` calls carry, and the bridge must be installed explicitly. Do not assume a dependency's logs appear unless you have wired this up.

**Compile-time and macro cost.** Heavy `#[instrument]` use expands to non-trivial code at every call site and can noticeably increase compile times and binary size in large codebases. The `max_level_*` and `release_max_level_*` Cargo features let you compile out verbose levels entirely in release builds — worth using for hot paths.

**Version skew across the ecosystem.** Because `tracing`, `tracing-subscriber`, `tracing-opentelemetry`, and dozens of third-party layers version independently, dependency graphs frequently pull two incompatible `tracing-subscriber` minor versions or an `opentelemetry` version that lags the tracing bridge. `tracing-opentelemetry` in particular tends to trail `opentelemetry` releases, and upgrading the OTel stack often blocks on it. Pin deliberately.

**The 0.2 rewrite that never shipped.** A `v0.2.x` branch containing a reworked `tracing-core` has existed for years and remains unreleased[^3]. In practice the entire ecosystem runs on 0.1, which has been kept stable and backward-compatible; treat 0.1 as the real, indefinitely-supported line rather than waiting for 0.2.

## When to Use / When Not

**Use when:**
- You are writing async Rust (Tokio, `async-std`) and need context to follow work across `.await` points.
- You want structured fields, per-module level filtering, and one instrumentation surface that can feed logs and distributed tracing (OpenTelemetry) together.
- You are authoring a library and want to emit diagnostics without imposing a logging backend on downstream users.

**Avoid / reconsider when:**
- You have a small synchronous CLI or script where the `log` + `env_logger` pair is enough — `tracing`'s model is overhead you will not use.
- Your team cannot absorb the conceptual cost of spans, layers, and the async-guard rule; misused, it produces worse output than plain logging.
- Binary size and compile time are hard constraints and you would lean heavily on macros.

## Alternatives

- rust-lang/log — the older, simpler logging facade; use it instead when you only need leveled string logs and no span/context model.
- tokio-rs/tracing-opentelemetry — not an alternative but the standard bridge; use it when you need OTLP/distributed traces out of `tracing`.
- open-telemetry/opentelemetry-rust — use directly when your primary need is vendor-neutral distributed tracing/metrics export and you don't want the `tracing` instrumentation model.
- slog-rs/slog — an earlier structured-logging library for Rust; use it when you want structured logging without spans and are not in an async-context-propagation situation.
- rust-lang/env_logger — a minimal `log` backend; pair with `log` for the simplest possible setup where `tracing` would be overkill.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-01-11 | Repository opened under tokio-rs[^4]. |
| tokio-rs/tracing 0.1.0 | 2019-10 | First 0.1 release of the `tracing` facade. |
| tracing-subscriber 0.1 | 2019 | Initial subscriber utilities crate. |
| tracing-subscriber 0.2 | 2020 | `Layer` trait and `Registry` composition model. |
| tracing-subscriber 0.3 | 2021 | Per-layer filtering (`Filter`); current line. |
| v0.2.x branch | ongoing | Reworked `tracing-core`; unreleased, ecosystem stays on 0.1[^3]. |
| last push | 2026-05-30 | Actively maintained; MSRV Rust 1.65[^1]. |

At the time of writing the repository has roughly 6,800 stars and 900 forks, with an open-issue count in the high hundreds — typical for a foundational, wide-surface project where "issues" include long-lived feature discussions and ecosystem tracking rather than only defects.

## References

[^1]: tokio-rs/tracing README and project overview. https://github.com/tokio-rs/tracing
[^2]: `tracing` documentation, "Entering a span" / closing spans and the async guard caveat. https://docs.rs/tracing/latest/tracing/span/index.html#closing-spans
[^3]: Branch set-up notes describing the unreleased `v0.2.x` line. https://github.com/tokio-rs/tracing/tree/v0.2.x
[^4]: GitHub repository metadata (created 2019-01-11, MIT license). https://github.com/tokio-rs/tracing

## Tags

rust, observability, logging, tracing, diagnostics, async, structured-logging, instrumentation, tokio, opentelemetry
