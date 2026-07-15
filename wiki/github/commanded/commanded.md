# commanded/commanded

> CQRS/event-sourcing framework for Elixir — command routing, aggregates as processes, event handlers, and process managers on the BEAM.

[GitHub repo](https://github.com/commanded/commanded) ·
[Hex package](https://hex.pm/packages/commanded) ·
[Docs](https://hexdocs.pm/commanded/) ·
[License: MIT](https://github.com/commanded/commanded/blob/main/LICENSE)

## Overview

Commanded is an Elixir framework for building applications with the Command Query Responsibility Segregation and event-sourcing (CQRS/ES) pattern[^1]. Rather than persisting current state, an event-sourced app persists the sequence of domain events that produced that state; Commanded supplies the moving parts around that idea — dispatching commands to aggregates, appending the resulting events to an event store, and running the handlers, projectors, and long-running process managers that react to those events. It was created by Ben Smith (slashdotdash), who also wrote the companion `eventstore` persistence library[^2].

The framework leans hard into OTP. Each aggregate instance is a GenServer process, command dispatch is serialized per aggregate, and process distribution across a cluster is handled by a pluggable registry. This makes it a natural fit for teams already committed to Elixir/Phoenix who want event sourcing to feel idiomatic on the BEAM rather than bolted on. It is not a general "add events to your CRUD app" library — adopting Commanded means adopting the full CQRS/ES model, with the read/write split and eventual consistency that implies.

The defining tension is that Commanded gives you a clean, well-factored programming model at the cost of real operational complexity: eventual consistency between the write and read sides, aggregates as single-writer processes that can become hot spots, and subscription/projection state that must be managed and occasionally rebuilt. The framework is honest about being a foundation, not a batteries-included product.

## Getting Started

Add Commanded plus an event store adapter to `mix.exs`. The most common pairing is `commanded_eventstore_adapter`, backed by the Postgres-based `eventstore` library[^2]:

```elixir
# mix.exs
def deps do
  [
    {:commanded, "~> 1.4"},
    {:commanded_eventstore_adapter, "~> 1.4"},
    {:eventstore, "~> 1.4"}
  ]
end
```

```elixir
# An aggregate: pure functions, no processes visible to you.
defmodule Account do
  defstruct [:id, balance: 0]

  # execute/2 returns events (or an error); it does not mutate state.
  def execute(%Account{}, %OpenAccount{id: id, initial: amt}) do
    %AccountOpened{id: id, initial: amt}
  end

  # apply/2 folds an event into the next state.
  def apply(%Account{} = acct, %AccountOpened{id: id, initial: amt}) do
    %Account{acct | id: id, balance: amt}
  end
end

defmodule BankApp do
  use Commanded.Application, otp_app: :bank_app

  router Commanded.Router
end

defmodule Commanded.Router do
  use Commanded.Commands.Router
  dispatch OpenAccount, to: Account, identity: :id
end
```

Dispatch a command with `BankApp.dispatch(%OpenAccount{id: "acc-1", initial: 100})`.

## Architecture / How It Works

Commanded splits into a few cooperating layers:

- **Router** — maps command structs to aggregate modules and declares how to derive the aggregate's identity (`identity: :field`). Composite routers let you split registration across bounded contexts.
- **Aggregates** — hosted one GenServer per aggregate instance, started lazily on first command and kept alive per a configurable lifespan. Your aggregate module stays pure: `execute/2` produces events, `apply/2` folds them into state. The process wrapper replays the event stream (optionally from a snapshot) to rebuild state before running a command, then serializes command execution so there is a single writer per instance. `Commanded.Aggregate.Multi` lets one command emit several events atomically.
- **Event store adapter** — Commanded talks to persistence through the `Commanded.EventStore` behaviour. Adapters exist for the Postgres `eventstore` library and an in-memory store meant only for tests[^3]. Appends are conditioned on expected stream version, giving optimistic concurrency.
- **Event handlers & projectors** — subscribe to the event stream (individually or via `Commanded.Projections.Ecto` for read models) and process events in order with at-least-once delivery. Each subscription tracks its position, so a handler resumes where it left off after a restart.
- **Process managers** — coordinate long-running workflows across aggregates, maintaining their own state and dispatching new commands in response to events (the saga pattern).

Consistency is configurable per dispatch. The default is `:eventual` — `dispatch/2` returns as soon as events are persisted, before handlers have processed them. `:strong` consistency makes dispatch block until all strongly-consistent handlers have caught up to the emitted events, which is what lets a Phoenix controller read its own write immediately at the cost of latency.

Process distribution is abstracted behind the `Commanded.Registration` behaviour. The default local registry runs everything on one node; clustered deployments swap in a distributed registry (historically Swarm, and Horde-based community adapters) so aggregate processes are singletons across the cluster.

## Production Notes

- **Aggregates are single-writer processes.** All commands for one aggregate instance queue through one GenServer. A "hot" aggregate (a shared counter, a popular account) serializes throughput and becomes a bottleneck — model aggregate boundaries so load spreads across many instances.
- **Eventual consistency is the default and it bites.** Code that dispatches a command and then immediately queries a read model will often see stale data. Either use `:strong` consistency on the dispatch (and accept the latency and timeout risk) or design the UI/API to tolerate the lag. Mixing the two carelessly is the most common source of "it works locally, flakes in prod" bugs.
- **Long streams need snapshots.** Without aggregate snapshotting, rehydrating an instance replays its entire event history on every cold start. Enable snapshots for aggregates whose streams grow without bound.
- **Rebuilding projections is a real operation.** Read-model schema changes usually mean resetting a projector's subscription and replaying from the start. Plan for downtime or double-write strategies; there is no free "just migrate the read model."
- **Event schema evolution.** Once events are persisted they are immutable. Renamed fields or restructured events must be handled with upcasting (`Commanded.Event.Upcaster`) rather than migrations.
- **Clustering was historically the rough edge.** Distributed registries built on Swarm could mishandle netsplits and process handoff under partition; the pluggable-registry design and Horde-based options exist partly in response. Test failover behaviour before relying on it.
- **Version floor.** Requires Erlang/OTP 21.0 and Elixir 1.11 or later[^1]. The 1.x line has kept a relatively stable public API since 1.0.

## When to Use / When Not

**Use when:**
- You are building on Elixir/Phoenix and genuinely want event sourcing — an audit-native domain, temporal queries, or complex workflows that suit process managers.
- Your domain decomposes into many independent aggregates so single-writer processes spread load naturally.
- You want the CQRS/ES model expressed idiomatically in OTP rather than assembled from primitives.

**Avoid when:**
- Your app is fundamentally CRUD; event sourcing adds eventual consistency and projection-rebuild overhead you will not get value from. Plain Ecto is simpler.
- You need synchronous read-after-write everywhere with low latency — the `:strong` consistency escape hatch has real cost.
- Your team is new to both Elixir and CQRS/ES simultaneously; the combined learning curve is steep and mistakes surface as data-modelling problems that are hard to undo.
- You only need durable event storage, not the full framework — the persistence library alone may be enough.

## Alternatives

- commanded/eventstore — the Postgres-backed event store beneath Commanded; use it directly when you want durable event streams and subscriptions without adopting the whole CQRS/ES framework.
- AxonFramework/AxonFramework — mature JVM CQRS/ES framework; use it when your stack is Java/Kotlin rather than the BEAM.
- EventStore/EventStore — EventStoreDB, a database-first, language-agnostic event store; use it when you want event sourcing driven from the storage engine outward across polyglot services.
- disco (and hand-rolled Ecto) — for lighter Elixir needs, an in-house `Ecto` + append-only table often beats a full framework when you only need a handful of event-sourced aggregates.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-03 | First public commits; framework grows alongside the `eventstore` library[^2]. |
| 0.x | 2017–2020 | Long 0.x series: routers, process managers, pluggable event-store adapters, distributed registry support. |
| 1.0 | 2021 | First stable major release; API stabilised, `Commanded.Application` introduced as the app entry point[^4]. |
| 1.4 | 2023–2026 | Current published line on Hex; ongoing fixes, upcasting, and adapter/registry refinements[^4]. |

## References

[^1]: Commanded README — CQRS/ES overview, feature list, and OTP/Elixir version requirements. https://github.com/commanded/commanded
[^2]: commanded/eventstore — Postgres-based Elixir event store by the same author, used as Commanded's default persistence backend. https://github.com/commanded/eventstore
[^3]: Choosing an Event Store — Commanded guides on adapters, including the in-memory store for tests only. https://hexdocs.pm/commanded/choosing-an-event-store.html
[^4]: Commanded changelog and published releases on Hex. https://hex.pm/packages/commanded

## Tags

elixir, cqrs, event-sourcing, ddd, beam, otp, framework, process-manager, eventstore, distributed-systems
