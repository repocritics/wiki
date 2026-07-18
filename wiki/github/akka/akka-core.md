# akka/akka-core

> The JVM's canonical actor-model toolkit — concurrency, clustering, and event
> sourcing for Scala and Java — now source-available under BSL 1.1, not open source.

[GitHub repo](https://github.com/akka/akka-core) ·
[Official website](https://doc.akka.io/libraries/) ·
[License: BUSL-1.1](https://github.com/akka/akka-core/blob/main/LICENSE)

## Overview

Akka is an implementation of the actor model for the JVM, started by Jonas Bonér
in 2009 and long stewarded by Typesafe/Lightbend. Actors are single-threaded
message-processing units with private state; Akka layers remoting, clustering,
sharding, event-sourced persistence, and a Reactive Streams engine (Akka Streams)
on top of that one primitive. It has both Scala and Java APIs and powered a large
share of the "reactive systems" era — Play Framework, Lagom, and many
finance/telecom/gaming backends sit on it.

Two facts dominate any current evaluation. First, in September 2022 Lightbend
relicensed Akka from Apache 2.0 to the Business Source License 1.1 starting with
2.7[^1][^2]: production use requires a paid license above a revenue threshold
(currently $25M/year), development and lower-revenue use are free, and each
release's source reverts to Apache 2.0 on a change date roughly three years
later. The move triggered one of the larger OSS forks of the decade: Apache
Pekko, a fork of Akka 2.6.x, which most open-source-dependent projects
(including Play) migrated to[^3]. Second, the company itself rebranded from
Lightbend to Akka and pivoted to a commercial "Akka SDK" and managed platform;
this repository was renamed from `akka/akka` to `akka/akka-core` and is now
positioned as the foundational library layer under that platform. The 13.3k
stars and 3.5k forks were mostly accrued in the Apache era; the last push
(2026-07) shows the core library is still actively maintained, with ~900 open
issues typical for a 17-year-old codebase of this scope.

## Getting Started

Since Akka 2.10, artifacts are distributed from Akka's own repository rather
than Maven Central, so a resolver entry is required[^4]:

```scala
// build.sbt
resolvers += "Akka library repository".at("https://repo.akka.io/maven")

val AkkaVersion = "2.10.0"
libraryDependencies += "com.typesafe.akka" %% "akka-actor-typed" % AkkaVersion
```

```scala
import akka.actor.typed.{ActorSystem, Behavior}
import akka.actor.typed.scaladsl.Behaviors

object Greeter {
  final case class Greet(name: String)

  def apply(): Behavior[Greet] =
    Behaviors.receiveMessage { msg =>
      println(s"Hello, ${msg.name}")
      Behaviors.same
    }
}

object Main extends App {
  val system: ActorSystem[Greeter.Greet] = ActorSystem(Greeter(), "hello")
  system ! Greeter.Greet("world")
}
```

## Architecture / How It Works

Everything reduces to actors scheduled on dispatchers. An `ActorRef` is a
location-transparent handle; a message send enqueues to the actor's mailbox, and
a dispatcher (backed by a fork-join or thread-pool executor) runs the actor's
receive function for a batch of messages. One actor never runs on two threads at
once, which is the whole concurrency model: no locks, no shared mutable state,
supervision trees for failure handling ("let it crash," borrowed from Erlang).

There are two actor APIs. **Classic** actors (untyped, `Any => Unit` receive)
are the 2009–2019 lineage. **Typed** actors (`Behavior[T]`, message protocols
checked at compile time) became the recommended API in 2.6 (2019)[^5]. Both
still ship and interoperate; most documentation and new features target typed.

The distributed stack is layered: **Artery** remoting (TCP or Aeron/UDP) →
**Cluster** (gossip membership, failure detection via phi-accrual) → **Cluster
Sharding** (partitions entity actors across nodes by shard key, with rebalancing)
→ **Cluster Singleton** and **Distributed Data** (CRDTs). **Akka Persistence**
implements event sourcing: actors persist events to a journal (Cassandra, JDBC,
R2DBC plugins) and rebuild state on replay. **Akka Streams** compiles a graph
DSL of sources/flows/sinks down to actors with demand-based backpressure; it is
a founding Reactive Streams implementation, and Akka HTTP (separate repo) is
built on it.

The coupling story: the libraries are individually published but designed as one
system — sharding needs cluster needs remoting needs a shared serialization
regime, and versions must be upgraded in lockstep across the Akka dependency
set. Adopting one distributed piece effectively adopts the platform.

## Production Notes

- **The license is the first production question.** Running Akka >= 2.7 in
  production above the revenue threshold requires a commercial subscription;
  CI/CD and dev use are free[^1]. Staying on Apache-licensed 2.6.x means no
  patches (including security) — that line is EOL. Pekko is the maintained
  Apache-2.0 escape hatch, with mostly mechanical `akka.*` → `pekko.*`
  migration for the 2.6-era API surface.
- **Blocking kills dispatchers.** A JDBC call or `Thread.sleep` inside an actor
  starves the default dispatcher and stalls unrelated actors. The standard fix
  is a dedicated bounded dispatcher for blocking work[^6]; this is the single
  most common Akka production incident pattern.
- **Serialization is config-heavy.** Java serialization has been disabled by
  default since 2.6; you must bind Jackson or protobuf serializers per message
  hierarchy, and rolling upgrades require wire-compatible message evolution.
  Persistence adds schema evolution of stored events — a long-term maintenance
  tax that teams routinely underestimate.
- **Split brain must be configured, not assumed.** A partitioned cluster will
  happily run two singletons or duplicate shards unless the Split Brain Resolver
  (commercial until it was open-sourced in 2.6.6, 2020) is enabled with a
  strategy matching your topology[^7]. Related: quarantined nodes cannot rejoin
  without a restart, which surprises operators during network flaps.
- **Unbounded mailboxes are the default.** Slow consumers show up as heap
  growth, not errors. Bounded mailboxes, streams backpressure, or explicit
  work-pulling are the mitigations.
- **Operational maturity is real.** Rolling updates, coordinated shutdown,
  Kubernetes discovery (Akka Management), and licensed multi-datacenter
  replication all exist and work; the tradeoff is that competent operation
  demands genuine understanding of the cluster protocol, not just YAML.

## When to Use / When Not

**Use when:**
- You need stateful, long-lived entities at scale (millions of digital twins,
  game sessions, IoT devices) and cluster sharding's entity-per-actor model fits.
- You are event sourcing on the JVM and want a mature persistence/projection
  stack rather than assembling one.
- You are already licensed (or under the revenue threshold) and value one
  coherent, battle-tested distributed toolkit over composing libraries.

**Avoid when:**
- BSL is a dealbreaker — policy, procurement, or open-source distribution.
  Use apache/pekko.
- Your services are stateless request/response; Spring Boot, Quarkus, or plain
  HTTP frameworks deliver the same outcome with far less conceptual load.
- Your team is Scala-averse and small: the Java API works, but the actor model,
  supervision, and cluster operations remain a steep, permanent learning cost.
- You only need structured concurrency in-process — Kotlin coroutines, Java
  virtual threads (Loom), or cats-effect/ZIO cover that without a cluster stack.

## Alternatives

- apache/pekko — the Apache-2.0 fork of Akka 2.6; use instead when you want the
  same API without BSL terms.
- microsoft/orleans — virtual actors on .NET; use when you want actor lifecycle
  managed implicitly rather than explicit supervision and placement.
- erlang/otp — the original actor runtime; use when you can adopt the BEAM and
  want preemptive scheduling and hot code upgrades.
- asynkron/protoactor-go — Go/.NET actors with gRPC transport; use for
  actor-model designs outside the JVM.
- zio/zio — typed effects and fibers for Scala; use when you need concurrency
  and streaming but not location transparency or clustering.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2009 | Initial public release by Jonas Bonér. |
| 2.0 | 2012-03 | Ground-up rewrite: location transparency, remoting-first design. |
| 2.3 | 2014-03 | Akka Persistence (event sourcing) introduced. |
| 2.5 | 2017-04 | Distributed Data stable; coordinated shutdown. |
| 2.6 | 2019-11 | Typed API becomes mainline; Artery default remoting; Java serialization off by default[^5]. |
| 2.6.6 | 2020-05 | Split Brain Resolver open-sourced into the core[^7]. |
| 2.7 | 2022-10 | License change Apache 2.0 → BSL 1.1[^2]; Pekko fork begins from 2.6.x[^3]. |
| 2.10 | 2024 | Artifacts move from Maven Central to repo.akka.io[^4]. |
| — | 2024–25 | Lightbend rebrands as Akka; repo renamed `akka/akka` → `akka/akka-core` under the Akka SDK platform pivot. |

## References

[^1]: Akka License FAQ (BSL 1.1 terms, revenue threshold, change dates). https://akka.io/bsl-license-faq
[^2]: Jonas Bonér, "Why We Are Changing the License for Akka" — 2022-09-07. https://www.lightbend.com/blog/why-we-are-changing-the-license-for-akka
[^3]: Apache Pekko — Apache-2.0 fork of Akka 2.6.x. https://pekko.apache.org/
[^4]: Akka Dependencies — current versions and the repo.akka.io repository. https://doc.akka.io/libraries/akka-dependencies/current/
[^5]: "Akka 2.6.0 released" — 2019-11-06. https://akka.io/blog/news/2019/11/06/akka-2.6.0-released
[^6]: Akka docs, "Dispatchers" — blocking management. https://doc.akka.io/libraries/akka-core/current/typed/dispatchers.html
[^7]: Akka docs, "Split Brain Resolver". https://doc.akka.io/libraries/akka-core/current/split-brain-resolver.html

## Tags

scala, java, jvm, actor-model, distributed-systems, concurrency, event-sourcing, streaming, cluster-computing, reactive, bsl-license
