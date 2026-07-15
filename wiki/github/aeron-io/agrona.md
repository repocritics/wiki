# aeron-io/agrona

> Off-heap buffers, primitive collections, and lock-free concurrency primitives for low-latency Java — the allocation-free toolkit underneath Aeron and SBE.

[GitHub repo](https://github.com/aeron-io/agrona) ·
[License: Apache-2.0](https://github.com/aeron-io/agrona/blob/master/LICENSE)

## Overview

Agrona is a library of data structures and utility methods for building high-performance JVM applications from Real Logic (Martin Thompson's group, the "mechanical sympathy" lineage)[^1]. It was extracted from the Aeron messaging codebase to hold the parts that were generically useful, and it still serves primarily as the shared foundation for Aeron and Simple Binary Encoding (SBE)[^2]. Maven coordinates are `org.agrona:agrona`; the GitHub org was renamed from `real-logic` to `aeron-io`, so `real-logic/agrona` now redirects here.

The defining goal is **allocation-free, GC-friendly execution on the hot path**. Everything in the library is built to avoid autoboxing, avoid per-operation object churn, and let you work directly with off-heap memory: primitive-keyed maps and sets that never box, array-backed primitive lists, flyweight buffers over on- or off-heap regions, single-writer lock-free queues and ring buffers, and off-heap counters for telemetry. The audience is narrow and self-selecting — trading systems, market-data handlers, and other latency-sensitive services where a stop-the-world GC pause is a defect, not an annoyance.

The central tension is that this performance comes from reaching under the JVM's safety guarantees. Agrona has historically depended on `sun.misc.Unsafe` for its buffer memory access, and it hands you APIs (unchecked buffers, reused iterators, sentinel-based "missing value" semantics) that are fast precisely because they trust the caller. Getting the guarantees wrong turns a would-be `IndexOutOfBoundsException` into a JVM crash. The library is also in the middle of a multi-year migration off `Unsafe` as the JDK deprecates it[^3].

## Getting Started

Gradle (`build.gradle.kts`):

```kotlin
dependencies {
    implementation("org.agrona:agrona:2.2.4")
}
```

Maven:

```xml
<dependency>
  <groupId>org.agrona</groupId>
  <artifactId>agrona</artifactId>
  <version>2.2.4</version>
</dependency>
```

A minimal example: a primitive-keyed map (no boxing) and a flyweight buffer over off-heap memory.

```java
import org.agrona.collections.Int2ObjectHashMap;
import org.agrona.concurrent.UnsafeBuffer;
import java.nio.ByteBuffer;

// Open-addressed, linear-probing map — keys stay primitive int, never Integer.
Int2ObjectHashMap<String> sessions = new Int2ObjectHashMap<>();
sessions.put(42, "active");
String state = sessions.get(42);          // no autoboxing on lookup

// Flyweight over a 1 KB direct (off-heap) buffer.
UnsafeBuffer buffer = new UnsafeBuffer(ByteBuffer.allocateDirect(1024));
buffer.putLong(0, 0xCAFEBABEL);           // little-endian by default
long value = buffer.getLong(0);
```

Agrona requires Java 17 or newer as of the 2.x line; it is tested against Java 17, 21, and 25[^4].

## Architecture / How It Works

Agrona is not a framework — it is a flat toolbox of independent components. The main groups:

- **Buffers.** `DirectBuffer` / `MutableDirectBuffer` are the flyweight interfaces; `UnsafeBuffer` is the workhorse implementation that can wrap a `byte[]`, a heap `ByteBuffer`, a direct `ByteBuffer`, or a raw off-heap address. `AtomicBuffer` adds ordered/volatile/CAS accessors with defined memory semantics. `ExpandableArrayBuffer` and `ExpandableDirectByteBuffer` grow on write for serialization use.
- **Collections.** Open-addressing, linear-probing maps and sets keyed on primitives: `Int2ObjectHashMap`, `Long2ObjectHashMap`, `Int2IntHashMap`, `Object2IntHashMap`, `IntHashSet`, plus `IntArrayList` / `LongArrayList` and a set-associative `Int2ObjectCache`. Missing entries are represented by a configurable sentinel `missingValue`, not `null` — a semantic you must respect.
- **Concurrency.** Lock-free `ManyToOneRingBuffer` / `OneToOneRingBuffer` and `BroadcastTransmitter` / `BroadcastReceiver` implemented off-heap for IPC; `ManyToOneConcurrentArrayQueue` and friends; and `IdleStrategy` implementations (`BusySpinIdleStrategy`, `BackoffIdleStrategy`, `YieldingIdleStrategy`, `SleepingIdleStrategy`, `NoOpIdleStrategy`) that let you trade CPU for latency explicitly.
- **Agent framework.** `Agent` + `AgentRunner` / `AgentInvoker` encode the single-threaded event-loop ("run-to-completion `doWork()`") pattern that Aeron itself uses, with `DynamicCompositeAgent` for composing several onto one thread.
- **Timers and IDs.** `DeadlineTimerWheel`, a hashed timer wheel with O(1) register and cancel; `SnowflakeIdGenerator`, a lock-free implementation of the Twitter Snowflake distributed-ID scheme.
- **Telemetry.** `CountersManager` / `AtomicCounter` maintain labelled off-heap counters for position tracking and coordination between processes.
- **Utilities.** `DistinctErrorLog` (deduplicates errors so a fault loop can't fill a disk), buffer-backed `InputStream`/`OutputStream`, and a small build-time code generator that specializes templates for primitive types.

Memory access has historically routed through `sun.misc.Unsafe`. Recent versions introduced an internal `UnsafeApi` indirection so the backing mechanism can be swapped as the JVM moves toward `VarHandle` and the Foreign Function & Memory (FFM) API — the ongoing structural change in the codebase[^3].

## Production Notes

- **Bounds checking is a runtime flag with teeth.** Buffer access is bounds-checked by default, but the checks can be disabled globally with `-Dagrona.disable.bounds.checks=true` for the last few percent of throughput. With checks off, an out-of-range index is undefined behavior against raw memory: expect a `SIGSEGV` and a hard JVM crash rather than an exception. Only disable it after the access patterns are proven correct.
- **`sun.misc.Unsafe` deprecation.** JDK 23+ emits warnings for the Unsafe memory-access methods Agrona relies on, and they are slated for removal[^3]. Running on recent JDKs may require flags such as `--sun-misc-unsafe-memory-access=allow` (plus native-access enablement) until the FFM migration is complete. Pin Agrona and JDK versions together and re-test on major JDK bumps.
- **Alignment matters for atomics.** The ordered/volatile/CAS accessors on `AtomicBuffer` assume natural alignment (e.g. 8-byte for `long`). Placing an atomically-accessed field at a misaligned offset is a correctness bug on some architectures, not just a slowdown.
- **Single-writer discipline.** Ring buffers and queues have distinct one-to-one vs many-to-one variants for a reason. Using a `OneToOne` structure with multiple producers silently corrupts state — the type is the contract.
- **Reused iterators and entries.** Collections reuse iterator and entry objects to stay allocation-free. Do not retain a reference to an entry across iteration steps, and do not hold iterators across threads.
- **`missingValue` semantics.** Primitive-valued maps return a sentinel for absent keys, not `null`. If a real value can equal the sentinel you will get false hits; choose the sentinel deliberately.
- **`BusySpinIdleStrategy` pins a core at 100% CPU.** It is the right choice for dedicated latency-critical threads on isolated cores, and the wrong choice in shared or cloud-throttled environments where it burns budget and starves neighbors. Pick the idle strategy to match the deployment, not just the latency target.
- **Direct buffers are not GC-managed memory.** Off-heap allocations you make yourself must be freed on the right lifecycle; leaking them leaks native memory that heap tuning won't surface.

## When to Use / When Not

**Use when:**
- You are building on Aeron or SBE — Agrona is already a transitive dependency and its buffers are the interchange type.
- You need allocation-free hot paths and primitive collections to keep GC out of tail latency.
- You want lock-free single-writer IPC (ring/broadcast buffers) or an off-heap counter/telemetry substrate.
- You are implementing a single-threaded event-loop service and want the agent/idle-strategy scaffolding done right.

**Avoid when:**
- You want a general-purpose collections library with a rich, safe API — Agrona's surface is deliberately spartan and sharp.
- Your workload is not latency-sensitive; the safety/ergonomics cost buys you nothing.
- You cannot commit to careful JDK/flag management around `Unsafe` deprecation.
- You need broad platform reach on old JVMs — the 2.x line is Java 17+ only.

## Alternatives

- eclipse/eclipse-collections — use instead when you want primitive and object collections with a broad, fluent, well-documented API for general application code.
- carrotsearch/hppc — use instead when you want just high-performance primitive collections without the buffers/concurrency machinery.
- JCTools/JCTools — use instead when you specifically need battle-tested lock-free concurrent queues and overlap little with Agrona's other components.
- OpenHFT/Chronicle-Bytes — use instead when you want off-heap buffers and serialization in the OpenHFT low-latency stack rather than the Aeron one.
- vigna/fastutil — use instead when you want the largest primitive-collection API surface and type coverage and don't need off-heap or concurrency primitives.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014 | Extracted from the Aeron codebase as a standalone utility library[^1]. |
| 1.x | 2018–2024 | Long-lived stable line; Java 8 baseline; `sun.misc.Unsafe`-backed buffers. |
| 2.0.0 | 2025 | Major bump: Java 8 support dropped, Java 17 baseline, `UnsafeApi` indirection toward FFM[^4]. |
| 2.2.x | 2025–2026 | Ongoing 2.x maintenance; tested against Java 17/21/25. |

The repository is actively maintained — commits land regularly and it moves in lockstep with Aeron and SBE releases.

## References

[^1]: Agrona README and repository, Real Logic / aeron-io. https://github.com/aeron-io/agrona
[^2]: Aeron and Simple Binary Encoding, the primary consumers of Agrona. https://github.com/aeron-io/aeron · https://github.com/aeron-io/simple-binary-encoding
[^3]: JEP 471, "Deprecate the Memory-Access Methods in sun.misc.Unsafe for Removal." https://openjdk.org/jeps/471
[^4]: Agrona Change Log (GitHub wiki). https://github.com/aeron-io/agrona/wiki/Change-Log

## Tags

java, jvm, low-latency, off-heap, high-performance, primitive-collections, lock-free, concurrency, ring-buffer, mechanical-sympathy, aeron
