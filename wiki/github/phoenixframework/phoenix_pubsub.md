# phoenixframework/phoenix_pubsub

> Distributed pub/sub and presence tracking for the BEAM — the messaging layer under every Phoenix Channel and LiveView.

[GitHub repo](https://github.com/phoenixframework/phoenix_pubsub) ·
[Official website](https://www.phoenixframework.org/) ·
[License: MIT](https://github.com/phoenixframework/phoenix_pubsub/blob/main/LICENSE.md)

## Overview

Phoenix.PubSub is the topic-based publish/subscribe library extracted from the Phoenix web framework in late 2015[^1]. It does two things: fan out messages to subscriber processes across an Erlang cluster (`Phoenix.PubSub`), and track which processes are present on which topics with an eventually-consistent CRDT (`Phoenix.Tracker`, the engine behind `Phoenix.Presence`). Every Phoenix application depends on it transitively — Channels, LiveView updates, and Presence all ride on it — which explains the mismatch between its modest 764 GitHub stars and its actual footprint: it is infrastructure people use daily without ever visiting the repo.

The defining tradeoff is that distribution comes "for free" only if you accept the BEAM's terms. PubSub piggybacks on Distributed Erlang: if your nodes are clustered, broadcasts reach every node with zero extra infrastructure — no Redis, no broker, no sidecar. If your nodes cannot form an Erlang cluster (some PaaS environments, cross-VPC setups), the default adapter silently degrades to single-node behavior and you must bolt on the separate Redis adapter. Delivery is fire-and-forget: at-most-once, unordered across senders, no persistence, no replay, no backpressure. It is a signaling fabric, not a message queue — a distinction that trips up teams who reach for it as one.

Maintenance is mature-and-quiet rather than active-and-churning: last push May 2026, 14 open issues, and a 2.x API that has been stable since 2020. For a foundational library, that is a feature.

## Getting Started

Add the dependency in `mix.exs` (already present in any Phoenix app):

```elixir
def deps do
  [{:phoenix_pubsub, "~> 2.0"}]
end
```

Start an instance under your supervision tree, then subscribe and broadcast:

```elixir
# application.ex
children = [
  {Phoenix.PubSub, name: MyApp.PubSub}
]

# any process (a LiveView, a GenServer, an IEx shell)
Phoenix.PubSub.subscribe(MyApp.PubSub, "user:123")
Phoenix.PubSub.broadcast(MyApp.PubSub, "user:123", {:user_updated, %{id: 123}})

# the subscriber receives it as a plain message
receive do
  {:user_updated, user} -> IO.inspect(user)
end
```

Messages arrive in the subscriber's mailbox like any Erlang message — in a LiveView you handle them in `handle_info/2`. `local_broadcast/3` skips remote nodes when you only need node-local fan-out.

## Architecture / How It Works

The library is two loosely coupled systems sharing a transport.

**PubSub core.** Since v2.0, local subscriptions live in a sharded Elixir `Registry` with duplicate keys[^2]: `subscribe/2` registers the caller's pid under the topic, and a local broadcast walks the registry partitions and `send/2`s each subscriber directly. There is no broker process on the hot path — the v1 design's per-node GenServer bottleneck was removed in the 2.0 rewrite that shipped alongside Phoenix 1.5 in April 2020[^3]. Remote delivery goes through an adapter behaviour (`Phoenix.PubSub.Adapter`). The default `Phoenix.PubSub.PG2` adapter uses Erlang process groups (`:pg` on OTP 23+, the deprecated `:pg2` before that) to locate the PubSub process on each remote node and sends one message per node; the receiving node then does its own local registry dispatch. Fan-out cost is therefore O(nodes) on the wire and O(local subscribers) per node.

**Custom dispatch.** `broadcast/4` accepts a dispatcher module implementing `dispatch/3`, and subscribers can attach metadata at subscribe time. Phoenix Channels exploit this for "fastlaning": the message is encoded to its wire format once, and the raw frame is handed to each websocket transport process, avoiding N redundant JSON encodings for N subscribers. This hook is available to any application, not just Phoenix.

**Tracker (presence).** `Phoenix.Tracker` maintains replicated presence state using an ORSWOT delta-CRDT (observed-remove set without tombstones)[^4]. Each node tracks its local processes and gossips deltas to peers via heartbeats over the PubSub transport itself. State is sharded across a configurable pool. Node failures are handled in two phases: a temporary "down" grace period (state retained), then a permanent-down cutoff after which the dead replica's state is dropped and, on rejoin, fully re-synced. The result is eventual consistency — two nodes can briefly disagree about who is online, by design.

Coupling story: PubSub has no dependency on Phoenix (usable in any Mix project), but it is hard-coupled to BEAM distribution semantics — full-mesh Distributed Erlang, node names, EPMD or an alternative discovery mechanism.

## Production Notes

- **Clustering is your problem.** PubSub does not form the cluster; it assumes one exists. Production deploys pair it with `libcluster` or the simpler `dns_cluster` for node discovery on Kubernetes/Fly.io. Forgetting this yields the classic symptom: everything works on one node, presence and broadcasts mysteriously miss users behind a load balancer with two.
- **No delivery guarantees.** Broadcasts to a node that is briefly unreachable are lost, not queued. After a netsplit heals, Tracker state converges (CRDT), but plain PubSub messages sent during the split are gone. Anything requiring durability, ordering, or acknowledgment belongs in a real queue.
- **No backpressure.** Broadcast is `send/2` under the hood. A slow subscriber accumulates an unbounded mailbox; a hot topic with thousands of local subscribers makes every broadcast pay that fan-out cost synchronously in the caller-side dispatch loop. Mitigations: shard topics (`"room:123"` not `"rooms"`), use `local_broadcast` where cross-node delivery isn't needed, and put an intermediary GenServer in front of expensive subscribers.
- **Full-mesh scaling ceiling.** Distributed Erlang meshes every node with every node; practical PubSub clusters run in the tens of nodes, not hundreds. Beyond that you need topology help (Partisan-style overlays or an external broker), which this library does not provide.
- **Tracker churn cost.** Presence gossip is cheap at steady state but grows with join/leave churn and cluster size. Very churny topics (large public rooms with fast-cycling anonymous users) can make heartbeat deltas a measurable share of inter-node traffic. Tune shard pool size and consider whether you need presence at all on such topics.
- **1.x → 2.0 upgrade** was breaking: adapter configuration moved out of the Endpoint config into an explicit supervision child, the adapter API changed, and the Redis adapter required a coordinated major bump of `phoenix_pubsub_redis`[^3]. Anything written against 2.x since 2020 has needed essentially no changes since — the API surface is small and frozen in practice.

## When to Use / When Not

**Use when:**
- You're on the BEAM and need intra-cluster fan-out (LiveView updates, chat, notifications, cache invalidation signals) without external infrastructure.
- You need "who is online" presence and can accept eventual consistency — Tracker is one of the few production-hardened CRDT presence implementations anywhere.
- You already run Phoenix: it is installed, supervised, and battle-tested in your stack whether you noticed or not.

**Avoid when:**
- You need durable, ordered, acknowledged, or replayable messaging — use a broker or a log, not pub/sub over process mailboxes.
- Consumers live outside the BEAM (polyglot services) — PubSub speaks Erlang messages, not a wire protocol.
- Your nodes cannot form a Distributed Erlang cluster and you don't want to operate the Redis adapter.
- You expect hundreds of nodes — full-mesh distribution won't carry you there.

## Alternatives

- phoenixframework/phoenix_pubsub_redis — official adapter; use when nodes can't cluster over Distributed Erlang but share a Redis.
- nats-io/nats-server — use when you need lightweight pub/sub across polyglot services, not just BEAM processes.
- rabbitmq/rabbitmq-server — use when you need durable queues, acks, and routing rather than fire-and-forget fan-out.
- redis/redis — use its built-in pub/sub when you already run Redis and need simple cross-language channels.
- elixir-lang/elixir — the standard-library `Registry` covers single-node local pub/sub with zero dependencies; PubSub only earns its keep once you cluster.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015-11 | Extracted from the Phoenix framework into a standalone library[^1]. |
| 1.0 | 2016-06 | Stable release used by Phoenix 1.2; PG2 and Redis adapters, per-node local server design. |
| 2.0 | 2020-04 | Rewrite shipped with Phoenix 1.5: Registry-based sharded local dispatch, new adapter behaviour, explicit supervision, `:pg` support[^3]. |
| 2.1 | 2022 | Maintenance series — current stable line; small fixes and OTP compatibility, no API changes of note. |

## References

[^1]: Repository created 2015-11-10, split out of phoenixframework/phoenix. https://github.com/phoenixframework/phoenix_pubsub
[^2]: `Phoenix.PubSub` documentation, HexDocs. https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html
[^3]: Chris McCord, "Phoenix 1.5.0 released" — 2020-04-22, notes the PubSub 2.0 migration. https://www.phoenixframework.org/blog/phoenix-1-5-final-released
[^4]: `Phoenix.Tracker` documentation (ORSWOT CRDT, replica lifecycle), HexDocs. https://hexdocs.pm/phoenix_pubsub/Phoenix.Tracker.html

## Tags

elixir, erlang, pubsub, presence, crdt, distributed-systems, real-time, phoenix, messaging, beam
