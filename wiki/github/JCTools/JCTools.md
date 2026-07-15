# JCTools/JCTools

> Java Concurrency Tools — the lock-free and wait-free concurrent queues the JDK never shipped.

[GitHub repo](https://github.com/JCTools/JCTools) ·
[Official website](http://jctools.github.io/JCTools) ·
[License: Apache-2.0](https://github.com/JCTools/JCTools/blob/master/LICENSE)

## Overview

JCTools is a small, focused Java library of concurrent data structures — overwhelmingly queues — that the `java.util.concurrent` package leaves out. The JDK ships `ConcurrentLinkedQueue` (unbounded, lock-free MPMC) and `ArrayBlockingQueue` (bounded, lock-based), but nothing that lets you say "I have exactly one producer and one consumer, give me the fastest possible bounded ring buffer." JCTools fills that gap by specializing on the producer/consumer *arity*: SPSC, MPSC, SPMC, and MPMC each get their own hand-tuned implementation[^1]. Started around 2014 by Nitsan Wakart (of the "psychosomatic, lobotomy, saw" performance blog), it has become the de-facto queue layer under much of the JVM's high-throughput infrastructure.

The defining design choice is that **arity is a compile-time contract, not a runtime check**. An `SpscArrayQueue` is wait-free and dramatically faster than a general MPMC queue precisely because it assumes — and does not defend against — a single producer and single consumer thread. Use it wrong (two producers on an SPSC queue) and you get silent corruption, not an exception. This is the central tension of the whole library: it trades the JDK's safe-by-default generality for raw throughput, and pushes the correctness burden onto the caller to pick the right structure for their actual thread topology.

JCTools is best understood as infrastructure-grade plumbing rather than an application dependency. It is used directly inside Netty, RxJava, Reactor, and numerous commercial systems[^2]; most Java developers consume it transitively without ever importing it. Its second defining trait is an obsession with **mechanical sympathy** — memory layout, false-sharing avoidance, and CPU cache behavior are first-class concerns, which is why so much of the codebase deals with field padding and `sun.misc.Unsafe`.

## Getting Started

Maven:

```xml
<dependency>
    <groupId>org.jctools</groupId>
    <artifactId>jctools-core</artifactId>
    <version>4.0.6</version>
</dependency>
```

A bounded single-producer/single-consumer ring buffer:

```java
import org.jctools.queues.SpscArrayQueue;

// capacity is rounded up to the next power of two internally
SpscArrayQueue<String> queue = new SpscArrayQueue<>(1024);

// Producer thread (exactly one):
if (!queue.offer("event")) {
    // queue full — offer never blocks, it returns false
}

// Consumer thread (exactly one):
String item = queue.poll();   // null when empty; poll never blocks
```

Batch draining via the expanded `MessagePassingQueue` interface, which avoids per-element method-call and volatile overhead:

```java
queue.drain(
    e -> process(e),                 // consumer
    idleCounter -> idleCounter + 1,  // wait strategy
    () -> !shuttingDown              // exit condition
);
```

## Architecture / How It Works

The library is organized around a matrix of **arity × boundedness × backing structure**:

- **Array-backed** (`SpscArrayQueue`, `MpscArrayQueue`, `MpmcArrayQueue`, `SpmcArrayQueue`) — bounded ring buffers over a single `Object[]`, capacity rounded up to a power of two so index wrapping is a mask, not a modulo.
- **Linked-array queues** (`SpscLinkedQueue`, `MpscLinkedQueue`, and the `...GrowableArrayQueue` / `...UnboundedArrayQueue` families) — hybrids that chain fixed-size array chunks. They combine the cache-friendliness of arrays with the unbounded capacity of linked lists, trading a chunk pointer follow at boundaries.
- **XAdd-based queues** (`MpscUnboundedXaddArrayQueue`, `MpmcUnboundedXaddArrayQueue`, added in 3.0[^3]) — producers claim slots with a single atomic `getAndAdd` (XADD) instead of a CAS retry loop, so producer cost stays flat under contention. Chunks are pooled to cut allocation.

**Field padding and false sharing.** The performance headline is cache-line management. Producer and consumer index fields are surrounded by padding fields so they land on separate 64-byte cache lines; without this, a producer's write to `producerIndex` would invalidate the cache line holding `consumerIndex` and stall the other thread ("false sharing"). Historically this padding was hand-written via class-inheritance layering; the code also uses `sun.misc.Unsafe` for direct, volatile-semantics field and array-element access that bypasses bounds checks.

**Unsafe vs Atomic vs Unpadded variants.** Because `sun.misc.Unsafe` is discouraged on modern JVMs and unavailable in some environments (Android, strict module systems), most queues ship in three flavors: the default `Unsafe` implementation, an `Atomic*` variant built on `AtomicLongFieldUpdater`/`AtomicReferenceArray` for portability, and `Unpadded*` variants that drop the false-sharing padding to shrink footprint when you have many small queues. The `Atomic` classes are generated from the `Unsafe` sources rather than maintained by hand.

**`MessagePassingQueue`** is the extended interface beyond `java.util.Queue`. It adds `relaxedOffer`/`relaxedPeek`/`relaxedPoll` (which may spuriously report full/empty in exchange for skipping some ordering guarantees) and the `drain`/`fill` batch methods. `poll()`/`offer()` returning `null`/`false` on empty/full — never blocking, never throwing — is the core behavioral contract.

## Production Notes

- **Pick the queue that matches your real thread topology.** The single biggest footgun: an `Mpsc*` queue is safe for many producers but *exactly one* consumer; `Spsc*` requires exactly one of each. There is no runtime guard. Two consumers polling an MPSC queue is undefined behavior — you will see dropped or duplicated elements, not a crash. Audit the actual threads, not the intended design.

- **`poll()` returning `null` is ambiguous by design.** `null` means "empty right now," but for the relaxed variants and under weak memory ordering it can transiently return `null` even when an element was just offered. Do not treat a single `null` as a durable "queue is drained" signal; use `drain`/`isEmpty` semantics or an explicit shutdown protocol.

- **`sun.misc.Unsafe` on JDK 16+.** The default queues reach for `Unsafe`. On recent JDKs this triggers "illegal reflective access" style warnings and is on a long-term deprecation path (JEP 471 targets `Unsafe` memory access for removal). JCTools degrades gracefully — it falls back to the `Atomic*` path when `Unsafe` is unavailable — but if you need to avoid `Unsafe` entirely, select the `Atomic*` classes explicitly rather than relying on the default types.

- **Capacity is rounded up to a power of two.** `new MpmcArrayQueue<>(1000)` does not give you 1000 slots; it rounds up (to 1024) so the index-to-slot mapping can use a bitmask. Size accounting and memory budgeting should assume the rounded value.

- **Padding costs memory.** Each padded queue carries dozens of unused long fields to isolate cache lines. That is negligible for a handful of long-lived queues and wasteful for thousands of short-lived ones — the case the `Unpadded*` variants exist to serve.

- **Runtime floor is old, build floor is not.** Compiled artifacts target Java 6-compatible bytecode, so the library runs on very old JVMs, but building from source requires JDK 8 and Maven. The published `jctools-core` jar is the only thing most users need; the benchmark, concurrency-test, and experimental modules are not meant for production consumption.

- **`jctools-experimental` is not stable API.** Implementations there may change signature or disappear between releases ("some may never graduate"). Depend only on `jctools-core`.

## When to Use / When Not

**Use when:**
- You have a known, fixed producer/consumer arity and need the lowest-latency queue for it (event loops, actor mailboxes, log/metrics pipelines, ring buffers between pinned threads).
- Allocation pressure and GC pauses matter and you want bounded or chunk-pooled queues.
- You are building framework-level infrastructure where a few nanoseconds and cache-line behavior are worth caring about.

**Avoid when:**
- Your producer/consumer counts are dynamic or unknown — a JDK `ConcurrentLinkedQueue` or `LinkedBlockingQueue` is safer and the arity guarantees can't be enforced.
- You need blocking semantics (put-when-full / take-when-empty). JCTools queues never block; you must build the wait strategy yourself.
- You want backpressure, fairness, or delivery guarantees at a higher level — reach for a full messaging/streaming library instead.
- The workload isn't hot enough to notice; the JDK queues are simpler and eliminate the whole class of "wrong arity" bugs.

## Alternatives

- LMAX-Exchange/disruptor — ring-buffer framework with a richer sequencing/barrier model; use it when you want a batteries-included event-processing pipeline rather than a raw queue primitive.
- java.util.concurrent (JDK built-ins) — use `ConcurrentLinkedQueue`/`LinkedBlockingQueue`/`ArrayBlockingQueue` when arity is dynamic or you need blocking, and want zero third-party dependencies.
- real-logic/agrona — similar mechanical-sympathy toolkit (ring/broadcast buffers, off-heap structures); use when you also need off-heap buffers and Aeron-style IPC primitives.
- JCTools inside Netty — Netty vendors JCTools MPSC queues internally; if you're already on Netty you may get the same structures transitively without a direct dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015-04-29 | First tagged release; core SPSC/MPSC/SPMC/MPMC array and linked queues[^1]. |
| 2.0 | 2016-10-28 | Second-generation API and packaging. |
| 2.1.2 | 2018-03-11 | Last of the 2.x line. |
| 3.0.0 | 2020-01-03 | XAdd-based unbounded linked-array queues; pooled chunks for reduced contention[^3]. |
| 3.3.0 | 2021-03-04 | Final 3.x release. |
| 4.0.1 | 2022-09-08 | 4.x line; `Unpadded` variants and continued Unsafe/Atomic split. |
| 4.0.5 | 2024-06-03 | Maintenance release. |
| 4.0.6 | 2026-02-26 | Latest release as of this writing; 5.0.0-SNAPSHOT in development. |

## References

[^1]: JCTools README and project description — "Java Concurrency Tools for the JVM," concurrent queues missing from the JDK. https://github.com/JCTools/JCTools
[^2]: JCTools documentation — listed consumers include Netty and RxJava. http://jctools.github.io/JCTools
[^3]: JCTools 3.0.0 release (2020-01-03) — XAdd-based unbounded linked array queues. https://github.com/JCTools/JCTools/releases

## Tags

java, jvm, concurrency, lock-free, wait-free, queues, data-structures, false-sharing, mechanical-sympathy, high-performance
