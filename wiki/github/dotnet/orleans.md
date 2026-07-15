# dotnet/orleans

> A virtual-actor framework for building distributed, stateful .NET services without writing the distribution plumbing yourself.

[GitHub repo](https://github.com/dotnet/orleans) ·
[Official website](https://docs.microsoft.com/dotnet/orleans) ·
[License: MIT](https://github.com/dotnet/orleans/blob/main/LICENSE)

## Overview

Orleans is a .NET framework for stateful distributed applications, built around
the *virtual actor model* it introduced at Microsoft Research[^1]. The unit of
computation is a **grain**: an object with a stable user-defined identity,
in-memory state, and single-threaded (turn-based) execution. Unlike classic
actors, grains are "virtual" — they always logically exist, are activated on
demand by the runtime, and are silently deactivated when idle. The developer
never creates, destroys, places, or locates an actor; you call `GetGrain<T>(id)`
and the runtime handles the rest[^2].

The framework was open-sourced by Microsoft in 2015 and moved into the `dotnet`
organization; it powers production backends inside Microsoft, most famously the
presence and matchmaking services for Halo and Gears of War[^3]. Its appeal is
that it lets teams experienced with single-server C# — objects, interfaces,
`async/await`, `try/catch` — write systems that shard state across a cluster
without hand-rolling partitioning, failover, or a distributed cache.

The defining tradeoff is abstraction leakage. The virtual-actor model hides an
enormous amount of distributed-systems machinery (membership, placement,
single-activation, message routing), and when that machinery misbehaves — split
clusters, transient double activations, a hot grain becoming a throughput
ceiling — you are debugging a system whose hard parts were deliberately made
invisible. Orleans is easy to start with and demands real operational
understanding to run at scale.

## Getting Started

```bash
dotnet add package Microsoft.Orleans.Server
dotnet add package Microsoft.Orleans.Sdk
```

```csharp
// Grain contract + implementation
public interface IHello : IGrainWithStringKey
{
    Task<string> SayHello(string name);
}

public class HelloGrain : Grain, IHello
{
    public Task<string> SayHello(string name) => Task.FromResult($"Hello, {name}");
}
```

```csharp
// Program.cs — .NET generic host with a single local silo (Orleans 7+)
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;

var builder = Host.CreateApplicationBuilder(args);
builder.UseOrleans(silo => silo.UseLocalhostClustering());

using var host = builder.Build();
await host.StartAsync();

var grains = host.Services.GetRequiredService<IGrainFactory>();
var hello = grains.GetGrain<IHello>("greeter");
Console.WriteLine(await hello.SayHello("world"));
```

For a real cluster, swap `UseLocalhostClustering()` for a clustering provider
(Azure Table, ADO.NET/SQL, Redis, Consul, DynamoDB, or Kubernetes) — this is how
silos discover each other and form a membership view.

## Architecture / How It Works

A **silo** is the host process that stores and runs grain activations; a group
of silos forms a **cluster**. Silos maintain a shared membership view backed by
an external store (the clustering provider) and use it to detect failures and
agree on who is alive. A grain call from a client or another grain is routed to
whichever silo currently holds that grain's activation, or triggers a placement
decision if none exists[^2].

Key runtime mechanics:

- **Single activation, single thread.** By default a grain has at most one
  activation cluster-wide, and that activation processes one request at a time
  (turn-based concurrency). This is what removes locks from application code —
  and what makes a popular grain a serialization bottleneck.
- **Placement.** When a grain activates, a configurable policy (random,
  prefer-local, activation-count/load-based, or custom) picks the silo.
- **Persistence.** Grain state is loaded into memory on activation and written
  back explicitly via `WriteStateAsync()`. There is no automatic write-behind;
  durability is the developer's responsibility.
- **Reminders and timers.** Timers are non-durable, in-memory callbacks;
  reminders are durable, storage-backed schedules that survive deactivation.
- **Streams, transactions, grain-call filters.** Managed pub/sub streams over
  queues (Event Hubs, Kinesis, etc.), decentralized distributed ACID
  transactions across grains, and interceptors for cross-cutting concerns.

Orleans 7.0 (2022) was a substantial rewrite of the internals: MSBuild-time IL
code generation was replaced with Roslyn source generators, and a new
version-tolerant serializer (`[GenerateSerializer]` / `[Id(n)]` attributes)
replaced the old wire format[^4]. Hosting also moved fully onto the .NET generic
host. This is the boundary that matters most for operators (see below).

## Production Notes

**Reentrancy and deadlocks.** Because grains are single-threaded and
non-reentrant by default, a call cycle (A → B → A) can deadlock: A holds its turn
waiting on B, which is waiting on A. Fixes are `[Reentrant]`, `[AlwaysInterleave]`
on specific methods, or breaking the cycle. This is the most common Orleans
footgun and it does not show up until concurrency rises.

**Hot grains cap throughput.** Single-activation + single-thread means one grain
is one logical CPU. Fan-in patterns (every request touches a global counter/index
grain) hit a hard ceiling no cluster size can fix. Use stateless workers,
sharding, or `[Reentrant]` for read-heavy grains.

**Single activation is not a hard guarantee.** Under network partitions or silo
failures the runtime can transiently produce two activations of the same grain
before membership converges. Do not rely on it for correctness where duplicate
processing would be catastrophic; use it for locality, not as a distributed lock.

**The 3.x → 7.0 upgrade is a migration, not a bump.** Namespaces, code generation,
and the serialization wire format all changed. A rolling upgrade across that
boundary generally is not possible because old and new silos cannot exchange
messages; teams typically stand up a new cluster and drain, or bridge with care.
Budget real time for it.

**Reminders are coarse and centralized.** Reminder granularity is roughly one
minute and every reminder lives in the reminder table; at high volume the table
and its ticking become a bottleneck. Reminders are for durable, low-frequency
scheduling — not a job queue.

**State writes are manual.** Forgetting `WriteStateAsync()` silently loses state
on deactivation. There is no dirty-tracking safety net by default.

## When to Use / When Not

**Use when:**
- You are on .NET and need many small, individually addressable stateful entities
  (players, devices/digital twins, sessions, carts, accounts).
- You want in-memory state with low latency plus managed failover, without
  building your own sharding/cache/partition layer.
- Your workload is naturally keyed and the per-key work fits single-threaded
  execution.

**Avoid when:**
- Your hot path is a few high-throughput aggregates — single-activation grains
  will bottleneck.
- You need a polyglot/language-agnostic actor layer (Orleans is .NET-only).
- Your problem is durable long-running workflows with replay/compensation — a
  workflow engine models that more directly than actors.
- You cannot invest in operating a stateful clustered system; a stateless service
  over a database is simpler.

## Alternatives

- akkadotnet/akka.net — classic (non-virtual) actor model for .NET; use when you
  want explicit actor lifecycle, supervision hierarchies, and location control.
- asynkron/protoactor-dotnet — cross-language virtual actors over gRPC; use when
  you need actors spanning .NET and Go.
- dapr/dapr — actors as one building block behind a language-agnostic sidecar;
  use when you want polyglot services and portable infra bindings over raw perf.
- temporalio/temporal — durable workflow orchestration; use when your core need
  is reliable multi-step workflows with replay, not stateful in-memory entities.
- microsoft/service-fabric — Reliable Actors on the Service Fabric platform; use
  when you are already committed to that hosting environment.

## History

| Version | Date | Notes |
|---------|------|-------|
| Research | 2014 | Virtual Actor model published by Microsoft Research[^1]. |
| Open source | 2015-01 | Released publicly under an OSI license. |
| 1.0 | 2015-10 | First stable release line. |
| 2.0 | 2018-10 | .NET Standard 2.0, DI-based configuration. |
| 3.0 | 2020-01 | Networking rewrite, distributed ACID transactions. |
| 7.0 | 2022-11 | Rewrite: source-gen codegen, version-tolerant serializer, generic host[^4]. |
| 8.0 | 2024-02 | Aligned with .NET 8. |
| 9.0 | 2024-11 | Aligned with .NET 9. |

## References

[^1]: P. Bernstein et al., "Orleans: Distributed Virtual Actors for Programmability and Scalability" — Microsoft Research Technical Report, 2014. https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/
[^2]: Microsoft Orleans documentation — "Grains" and runtime overview. https://docs.microsoft.com/dotnet/orleans
[^3]: Microsoft Research Orleans project home (production usage, incl. Halo services). http://research.microsoft.com/projects/orleans/
[^4]: Microsoft Orleans 7.0 — serialization and code-generation changes. https://docs.microsoft.com/dotnet/orleans/host/configuration-guide/serialization

## Tags

csharp, dotnet, actor-model, virtual-actors, distributed-systems, cloud-native, stateful-services, concurrency, microsoft, grains, clustering
