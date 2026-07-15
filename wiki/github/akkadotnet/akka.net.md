# akkadotnet/akka.net

> A .NET port of the JVM Akka toolkit — the actor model, remoting, and clustering for C# and F#.

[GitHub repo](https://github.com/akkadotnet/akka.net) ·
[Official website](https://getakka.net) ·
[License: Apache-2.0](https://github.com/akkadotnet/akka.net/blob/dev/LICENSE)

## Overview

Akka.NET is an idiomatic .NET reimplementation of the Akka toolkit originally built for the JVM in Scala/Java[^1]. It provides the actor model — lightweight objects that communicate only by asynchronous message passing, process one message at a time, and encapsulate mutable state without locks — plus the higher layers built on top of it: `Akka.Remote` (location-transparent messaging across processes), `Akka.Cluster` (peer-to-peer membership with gossip and failure detection), `Akka.Cluster.Sharding` (distributing stateful entities across nodes), `Akka.Persistence` (event sourcing), and `Akka.Streams` (backpressured stream processing). The project began around 2013 and is a .NET Foundation member[^2].

The audience is teams building stateful, concurrent, or distributed systems in .NET — trading platforms, IoT backends, real-time simulation, multiplayer game servers — where the alternative would be hand-rolled locking, queues, and consensus. The defining tension is that Akka.NET buys you a battle-tested distributed-systems substrate at the cost of a large conceptual surface and a framework that owns your application's threading and lifecycle. It is not a library you call; it is a runtime you build inside.

A second tension is provenance. Akka.NET tracks a JVM project it does not control, and the upstream Akka relicensed to the BSL (Business Source License) in 2022. Akka.NET remains Apache-2.0 and is governed independently under the .NET Foundation, but the two codebases have diverged and Akka.NET's roadmap is now set by its own maintainers (primarily Petabridge) rather than shadowing JVM Akka[^3]. GitHub's license detector reports `NOASSERTION` for the repo, but the `LICENSE` file is Apache-2.0.

## Getting Started

```
dotnet add package Akka.Hosting
```

`Akka.Hosting` is the recommended entry point — it wires the actor system into the `Microsoft.Extensions` host, logging, configuration, and DI. The base `Akka` package can be used standalone.

```csharp
using Akka.Actor;

public record Greet(string Who);

public class Greeter : ReceiveActor
{
    public Greeter() =>
        Receive<Greet>(msg => Sender.Tell($"Hello, {msg.Who}"));
}

using var system = ActorSystem.Create("demo");
var greeter = system.ActorOf<Greeter>("greeter");

// Ask returns a Task; Tell is fire-and-forget
var reply = await greeter.Ask<string>(new Greet("world"), TimeSpan.FromSeconds(3));
Console.WriteLine(reply); // Hello, world
```

## Architecture / How It Works

The unit of execution is the actor, addressed by an `IActorRef` handle rather than a direct reference. Messages sent to a ref are enqueued in the actor's mailbox and dispatched one at a time onto a shared thread pool (the `Dispatcher`). Because only one message is in flight per actor, actor-internal state needs no synchronization. Actors form a supervision hierarchy: every actor has a parent that decides — via a `SupervisorStrategy` — whether to restart, resume, stop, or escalate on child failure. "Let it crash" is the intended fault model; you design restarts, not defensive try/catch around every message.

`IActorRef` is location-transparent. The same `Tell` call works whether the target is local or on another node; `Akka.Remote` serializes the message and ships it over the wire. This transparency is the core abstraction and also the core footgun: a local send is cheap and lossless, a remote send is neither, and the API hides the difference. Remoting historically used a bespoke transport (Helios, later DotNetty); newer versions default to `Akka.Remote.Transport.Tcp` / the Artery-style transport, and serialization defaults have moved from the JSON-based `NewtonsoftJson` serializer toward faster binary options for high-throughput paths.

Above remoting, `Akka.Cluster` maintains membership through gossip and phi-accrual failure detection, with no master node. `Cluster.Sharding` uses this to place and rebalance stateful entity actors across the cluster by entity id, and `Distributed Data` provides CRDT-based replicated state. `Akka.Persistence` gives actors durable event-sourced state via pluggable journals (SQL Server, PostgreSQL, MongoDB, Azure, etc.), and `Akka.Streams` layers a typed, backpressured dataflow API over actors implementing the Reactive Streams protocol.

Configuration is pervasive and uses HOCON, inherited from the JVM project[^4]. Nearly every subsystem — dispatchers, mailboxes, serializers, remoting endpoints, cluster seed nodes — is tuned through HOCON, which is powerful but opaque to newcomers. `Akka.Hosting` exists largely to replace HOCON strings with typed C# builders.

## Production Notes

- **HOCON is the hidden difficulty.** Most "it doesn't work in the cluster" problems trace to misconfigured HOCON: wrong hostname/public-hostname in remoting, missing seed nodes, serializer bindings not matching between nodes. Prefer `Akka.Hosting`'s typed configuration to avoid stringly-typed mistakes.
- **Serialization is a versioning liability.** The default JSON serializer is convenient but slow and fragile across message-contract changes. For remoting/persistence at scale, teams move to a versioned binary serializer and treat messages and persisted events as a schema that must evolve compatibly — a rename can break replay of old events.
- **`Ask` is not free.** Every `Ask` allocates a temporary actor and a timeout. Overusing `Ask` in hot paths (instead of `Tell` with explicit reply protocols) is a common throughput and GC problem.
- **Cluster formation and split-brain.** A cluster must reach the configured seed nodes to form, and network partitions can strand nodes as `Unreachable`. Production clusters need a Split Brain Resolver strategy configured explicitly; older deployments that relied on manual downing risked two halves both believing they were the surviving cluster.
- **Upgrade friction.** The v1.4 → v1.5 transition changed serialization and transport defaults and deprecated the legacy Helios/DotNetty paths; rolling upgrades across a live cluster require wire-compatible serializers on both old and new nodes during the overlap. Read migration guides before a rolling deploy — mixed-version clusters are where breakage surfaces.
- **Testing.** `Akka.TestKit` (with xUnit/NUnit bindings) is essential; testing actor systems without it tends toward flaky `Thread.Sleep`-based assertions. MultiNode tests for cluster behavior are slow and infrastructure-heavy.

## When to Use / When Not

**Use when:**
- You have genuinely stateful, concurrent domain logic (per-user, per-device, per-order actors) that maps cleanly to the entity-per-actor model.
- You need location transparency, cluster sharding, or event sourcing and want a maintained .NET implementation rather than assembling them from parts.
- Your team can invest in learning the supervision, addressing, and HOCON model.

**Avoid when:**
- The workload is stateless request/response — ASP.NET Core with a database is simpler and the actor overhead buys nothing.
- You want a small dependency; Akka.NET is a framework that shapes your whole application.
- You need one-off background work or channels — `System.Threading.Channels`, `IHostedService`, or a message queue are lighter.
- The team is unfamiliar with distributed-systems failure modes and cannot budget for the learning curve.

## Alternatives

- microsoft/orleans — "virtual actor" model with automatic activation/GC of grains; use instead when you want the runtime to manage actor lifecycle for you rather than supervising it explicitly.
- dotnet/aspnetcore — use instead when the workload is stateless HTTP and actors add complexity without payoff.
- dotnet/orleans (Grains) and Dapr actors — use Dapr when you want a language-agnostic sidecar actor runtime decoupled from .NET internals.
- akka/akka — the JVM original; use when you are on the JVM and can accept the BSL license terms.
- proto-actor — cross-platform, gRPC/Protobuf-based actor framework; use when you want lighter, protocol-first remoting and multi-language actors.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013-12 | Repository created; early port of JVM Akka to .NET[^2]. |
| 1.0 | 2015-04 | First stable release; core actors, remoting, clustering. |
| 1.3 | 2017 | .NET Standard support; Akka.Streams and persistence maturing. |
| 1.4 | 2020-03 | .NET Core / .NET Standard 2.0 baseline, streams stabilized. |
| 1.5 | 2023 | New serialization/transport defaults, Akka.Hosting emphasis, DotNetty deprecation path[^3]. |

## References

[^1]: Akka.NET README and getakka.net — ".NET implementation of the actor model built on the CLR." https://getakka.net/
[^2]: Akka.NET is a .NET Foundation project; repository created 2013-12-29 per GitHub API. https://dotnetfoundation.org/
[^3]: Petabridge, maintainer of Akka.NET, on the v1.5 direction and independence from JVM Akka. https://petabridge.com/blog/
[^4]: HOCON configuration format, inherited from the JVM Akka/Lightbend Config project. https://getakka.net/articles/configuration/config.html

## Tags

csharp, dotnet, actor-model, concurrency, distributed-systems, clustering, event-sourcing, reactive, fsharp, message-passing, akka
