# whitfin/cachex

> The de facto standard in-memory caching library for Elixir — ETS underneath, with TTLs, transactions, fallbacks, and optional cluster partitioning on top.

[GitHub repo](https://github.com/whitfin/cachex) ·
[Official docs](https://hexdocs.pm/cachex/) ·
[License: MIT](https://github.com/whitfin/cachex/blob/main/LICENSE)

## Overview

Cachex is an in-process key/value cache for Elixir applications, started by Isaac Whitfield in early 2016 and continuously maintained since[^1]. Every feature beyond get/put — time-based expiration, maximum-size limits, pre/post execution hooks, proactive warmers, transactions with row locking, filesystem persistence, statistics — is opt-in and off by default, so a bare cache is close to a supervised ETS table with a friendlier API.

Its defining tradeoff is being *in-process*. Data lives in an ETS table inside your BEAM node: reads are microsecond-fast with no serialization or network hop, but the cache dies with the node and, in a cluster, each node sees its own copy unless you opt into Cachex's distribution layer — which partitions keys across nodes rather than replicating them[^3]. Teams expecting Redis-like shared, durable semantics from the feature list are the most common source of misplaced expectations.

At roughly 1.7k stars and 123 forks it is one of the larger libraries in the (small) Elixir ecosystem, and the default answer to "how do I cache in Elixir" in most community threads. Maintenance is real but slow-cadence and effectively single-maintainer: v4.0.0 shipped September 2024, v4.1.1 in November 2025, and the last push to `main` was February 2026[^4]. Mature and stable rather than fast-moving.

## Getting Started

```elixir
# mix.exs
def deps do
  [{:cachex, "~> 4.0"}]
end
```

Start a cache under your application's supervision tree (recommended over ad-hoc `start_link`):

```elixir
# lib/my_app/application.ex
children = [
  {Cachex, [:my_cache]}
]
```

Basic usage:

```elixir
:ok = Cachex.put(:my_cache, "key", "value", expire: :timer.minutes(5))
"value" = Cachex.get(:my_cache, "key")

# lazy computation on miss — concurrent misses are deduplicated
{:commit, user} = Cachex.fetch(:my_cache, user_id, fn id ->
  {:commit, Repo.get(User, id)}
end)
```

Fallible calls return `{:error, reason}` tuples; every function has a `!` variant that unwraps or raises, intended for tests rather than production code paths.

## Architecture / How It Works

Each cache is its own supervision tree. The backing store is a single ETS table owned by a dedicated process (via the same author's `eternal` library) so the table survives crashes of processes that use it. Around the table sit a handful of named services:

- **Courier** — serializes `fetch/4` fallback execution per key, so a thundering herd of concurrent misses triggers exactly one computation and all callers receive the result. This is the built-in cache-stampede protection.
- **Janitor** — periodic sweep that purges expired records. Expiration is otherwise *lazy*: a read of an expired key treats it as missing, but the record occupies memory until the Janitor's next pass (interval configurable).
- **Locksmith** — implements transactions and row locking. Only keys named in a `Cachex.transaction/3` call are locked; untouched keys proceed at full ETS speed. Transactional writes serialize through this service, which is where the throughput cost lives.
- **Informant / hooks** — hooks are processes implementing a behaviour, notified before or after cache actions. Statistics gathering and size-limit enforcement are themselves implemented as hooks, not core code.

Version 3 (2018) was a ground-up rewrite that removed the Mnesia-backed transaction layer of v1/v2 in favor of the pure-ETS Locksmith design, which substantially improved throughput[^2]. Version 4 (2024) rewrote distribution around pluggable **routers** — modulo, jump-hash, and `libring`-based ring routing — that decide which connected node owns each key, with cross-node calls dispatched transparently[^3]. Size limits in v4 are enforced by LRW (least-recently-*written*) trim policies, available in scheduled (periodic) or evented (hook-triggered) forms.

## Production Notes

- **Limits are approximate and count entries, not bytes.** The limit policy trims a configurable fraction of entries once the count crosses the threshold, and enforcement is asynchronous — the cache can overshoot between checks. If you need a hard memory ceiling, you must derive an entry budget from your own value-size estimates. Eviction order is by write time (LRW), not access time: hot-but-old entries get evicted.
- **Expired data lingers between Janitor sweeps.** Reads are correct (lazy expiration), but memory usage and raw entry counts include expired-but-unswept records. Disabling the Janitor without understanding this leads to slow memory growth.
- **Transactions serialize.** Row locking routes writes through the Locksmith; heavy transactional workloads bottleneck there. The project's own benchmark suite gates transactional benchmarks behind an env flag precisely because the profile differs so much. Use `get_and_update/3` or `incr/2` for single-key atomicity instead of full transactions where possible.
- **Slow hooks stall the cache.** Synchronous pre-hooks sit in the call path of every action. Keep hooks asynchronous unless you specifically need to block.
- **Distribution is partitioning, not replication.** A router assigns each key to one node; losing a node loses that partition outright. There is no rebalancing story for the modulo/jump routers when membership changes — only the ring router is designed for dynamic clusters. Multi-key operations are constrained to keys residing on the same node. If you need replication or durability, this is not the tool.
- **v3 → v4 upgrade** renamed `dump`/`load` to `save`/`restore`, removed the standalone fallback option in favor of `fetch`, reworked hook and warmer definitions, and replaced the old `nodes:` distribution option with routers. Mechanical but nontrivial; a migration guide is maintained in the docs[^5].

## When to Use / When Not

**Use when:**
- You want per-node caching of expensive computations or DB reads in an Elixir/Phoenix app, with TTLs and stampede protection out of the box.
- You need cache features (warming, persistence snapshots, stats, limits) without operating an external service.
- Read latency matters: in-process ETS reads beat any network cache by orders of magnitude.

**Avoid when:**
- You need a shared cache with strong cross-node consistency or survival across deploys — use Redis/Valkey or a database.
- You need replication or high availability of cached data; Cachex's distribution partitions and offers no redundancy.
- A raw `:ets` table or `:persistent_term` suffices (e.g., read-mostly config); Cachex's supervision and option layers add overhead you may not need.

## Alternatives

- cabol/nebulex — caching *framework* with swappable adapters (local, partitioned, replicated, Redis) and declarative decorators; use it when you need replication or want to change backends without touching call sites.
- sasa1977/con_cache — much smaller ETS wrapper with TTLs and dirty/isolated operations; use it when Cachex's feature surface is overkill.
- redix (whatyouhide/redix) — Redis client; use an out-of-process Redis/Valkey cache when data must be shared across nodes and survive restarts.
- Built-in `:ets` / `:persistent_term` — no dependency at all; use for static or read-mostly data where TTLs and eviction are unnecessary.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2016-04 | First stable release[^4]. |
| 2.0.0 | 2016-10 | Second-generation API; Mnesia-era transaction layer. |
| 3.0.0 | 2018-02 | Ground-up rewrite: pure ETS, Locksmith locking, `fetch`-style fallbacks[^2]. |
| 3.6.0 | 2023-02 | Final minor of the v3 line. |
| 4.0.0 | 2024-09 | Router-based distribution, `save`/`restore` rename, hook/warmer rework[^5]. |
| 4.1.1 | 2025-11 | Latest release as of this page[^4]. |

## References

[^1]: Cachex documentation. https://hexdocs.pm/cachex/
[^2]: Cachex migration guide, v2 → v3. https://hexdocs.pm/cachex/migrating-to-v3.html
[^3]: Cachex docs (guides on distributed caches and routers, kept in-repo). https://github.com/whitfin/cachex/tree/main/docs
[^4]: Cachex GitHub releases. https://github.com/whitfin/cachex/releases
[^5]: Cachex migration guide, v3 → v4. https://hexdocs.pm/cachex/migrating-to-v4.html

## Tags

elixir, caching, in-memory, key-value, ets, ttl, distributed-systems, beam, performance, memory-cache
