# LMAX-Exchange/disruptor

> A bounded, lock-free ring buffer for passing messages between threads in a single JVM — the reference implementation of "mechanical sympathy" concurrency.

[GitHub repo](https://github.com/LMAX-Exchange/disruptor) ·
[Official website](https://lmax-exchange.github.io/disruptor/) ·
[License: Apache-2.0](https://github.com/LMAX-Exchange/disruptor/blob/master/LICENCE.txt)

## Overview

The Disruptor is a Java library for inter-thread messaging, extracted from LMAX's retail financial trading platform and open-sourced in 2011[^1]. Its purpose is narrow: move events between producer and consumer threads inside one process at the lowest possible latency, without the lock contention, allocation, and cache traffic that a `BlockingQueue` incurs. It is not a distributed queue, a broker, or a persistence layer.

The design grew out of the observation that a conventional bounded queue has two ends contended by different threads, and its head/tail counters live on shared cache lines that ping-pong between cores. LMAX needed to process on the order of millions of orders per second on a single business-logic thread[^2], and found that queues, not the business logic, were the bottleneck. The Disruptor replaces the queue with a pre-allocated ring buffer and a set of monotonic sequence counters, so producers and consumers coordinate by reading each other's published position rather than by locking.

The defining tension is generality versus speed. The Disruptor is faster than `java.util.concurrent` queues under contention, but it is a framework, not a drop-in `Queue`: you accept a fixed-size buffer, a power-of-two constraint, mutable reused event objects, and an explicit dependency graph of consumers. Used correctly it is very fast; used as a naive queue replacement it leaks stale event state and surprises operators with pinned CPU cores. It won a Duke's Choice Award in 2011[^3] and its pattern was later adopted by Log4j 2's asynchronous loggers, among others.

## Getting Started

Gradle:

```groovy
implementation 'com.lmax:disruptor:4.0.0'
```

Maven:

```xml
<dependency>
  <groupId>com.lmax</groupId>
  <artifactId>disruptor</artifactId>
  <version>4.0.0</version>
</dependency>
```

Minimal single-producer / single-consumer pipeline using the DSL:

```java
// Event carried through the ring buffer — reused, never re-allocated.
class LongEvent { long value; }

int bufferSize = 1024;               // MUST be a power of two
Disruptor<LongEvent> disruptor = new Disruptor<>(
        LongEvent::new,              // factory: pre-fills every slot
        bufferSize,
        DaemonThreadFactory.INSTANCE);

// Consumer: (event, sequence, endOfBatch)
disruptor.handleEventsWith((event, seq, endOfBatch) ->
        System.out.println("consumed " + event.value));

RingBuffer<LongEvent> ringBuffer = disruptor.start();

// Producer: claim a slot, mutate it in place, publish.
ringBuffer.publishEvent((event, seq, input) -> event.value = input, 42L);
```

## Architecture / How It Works

The core is a fixed-size array of event objects — the **ring buffer** — allocated once at startup. Because size is a power of two, the slot for sequence `n` is `n & (size - 1)`, a bitmask instead of a modulo. Events are never created per message; producers overwrite the object already sitting in the slot.

Coordination is done entirely through **sequences**: monotonically increasing `long` counters. A producer claims the next sequence, writes into the corresponding slot, then publishes by advancing the cursor. Each consumer holds its own sequence and processes everything up to the highest published position it is allowed to see. There are no locks on the hot path; publication and consumption are ordered by memory barriers around volatile sequence reads and writes.

**Sequence barriers** encode the dependency graph. Consumers can be wired so that one runs only after another has processed an event (for example, a journaling consumer before a business-logic consumer). The DSL's `handleEventsWith` / `then` / `after` builds this graph, and the barrier ensures a downstream consumer never overtakes its upstream dependency without a queue between them.

**False-sharing avoidance** is explicit: the `Sequence` value is padded so that the hot counter occupies its own cache line, preventing the cursor from ping-ponging with unrelated fields on adjacent lines. This padding is the mechanical-sympathy detail the library is known for.

**Wait strategies** are the main latency/CPU knob. A consumer with no new events must decide how to wait:

- `BusySpinWaitStrategy` — spins on the CPU; lowest latency, burns a full core.
- `YieldingWaitStrategy` — spins then `Thread.yield()`; low latency, high CPU.
- `SleepingWaitStrategy` — spins then parks briefly; balanced.
- `BlockingWaitStrategy` — lock/condition; lowest CPU, highest latency; the safe default for shared hosts.
- `PhasedBackoffWaitStrategy` — escalates through the above.

**Producer type** is chosen at construction: single-producer mode is faster because it skips the CAS needed to arbitrate multiple producers claiming sequences. When a consumer falls behind, it naturally catches up in **batches** — it reads all available published events in one pass — so throughput rises under load instead of collapsing.

## Production Notes

**Reused event objects are the classic footgun.** Every slot's object is mutated in place and lives forever. A handler that only sets some fields will read stale values left by a previous event thousands of iterations ago. Either fully overwrite every field on publish, or copy data out of the event before yielding; never hold a reference to the event object past the handler call.

**Busy-spin strategies pin cores at 100%.** `BusySpinWaitStrategy` and `YieldingWaitStrategy` are built for dedicated, thread-pinned hardware (the HFT use case). On shared cloud instances or containers with CPU limits they waste an entire vCPU per waiting consumer and distort autoscaling. Use `BlockingWaitStrategy` unless you own the box and have measured the latency benefit.

**The buffer is bounded and fixed at construction.** Size must be a power of two and cannot grow. Sizing is a real decision: too small and producers stall under bursts; too large and you hold a big array of live references, adding GC-root pressure that partly offsets the allocation savings. Backpressure (producers blocking when the ring is full) is by design, but your producer code must tolerate it.

**Exception handling defaults are quiet.** A handler that throws goes through the configured `ExceptionHandler`; the default logs and continues, but a misconfigured or fatal handler can halt a consumer and silently stall the pipeline behind it. Set an explicit `setDefaultExceptionHandler` and alert on it.

**4.0 is a hard break.** Version 4.0.0 (2023) requires Java 11, ships as a JPMS module (`com.lmax.disruptor`), and removed long-deprecated 3.x APIs[^4]. Upgrading from 3.x is not a version bump — expect source changes. If you are stuck on Java 8, the 3.4.x line is the end of the road.

**It is stable to the point of "done."** Releases are infrequent and the issue count is low; this is mature, narrow-scope software, not an actively evolving framework. Treat it as a settled dependency, and do not expect new features. Ports to other languages exist but are unofficial and lag the Java original.

## When to Use / When Not

**Use when:**
- You have an ultra-low-latency, high-throughput event pipeline inside a single JVM (market data, order matching, real-time risk, async logging).
- Lock contention or GC pauses from a `BlockingQueue` are a measured bottleneck.
- Your stages form a known, bounded dependency graph and you control both producers and consumers.
- You can dedicate cores and tune a wait strategy.

**Avoid when:**
- You need messaging across processes or machines — this is in-JVM only.
- You need durability, replay, or persistence.
- Throughput is not your bottleneck; `ArrayBlockingQueue` is simpler and correct.
- You run on shared/elastic infrastructure where pinning cores for spin-waits is wasteful.

## Alternatives

- real-logic/aeron — use instead when you need low-latency messaging across processes or over the network, not just between threads.
- OpenHFT/Chronicle-Queue — use instead when you need a persisted, off-heap, replayable queue rather than a volatile in-memory buffer.
- JCTools/JCTools — use instead when you want faster lock-free queue primitives without adopting the ring-buffer + consumer-graph framework.
- real-logic/agrona — use instead when you want lighter ring-buffer and data-structure building blocks to assemble yourself.
- java.util.concurrent (ArrayBlockingQueue) — use instead when throughput and latency are not the constraint and standard-library simplicity wins.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2011 | Open-sourced from LMAX; ring buffer + sequences[^1]. |
| 2.0 | 2011 | API overhaul; sequence/barrier model refined. |
| 3.0 | 2013 | Rewrite; the `Disruptor` DSL and `EventHandler` model matured. |
| 3.4 | 2017 | Long-lived 3.x line; final Java 8-compatible series. |
| 4.0 | 2023 | Requires Java 11, JPMS module, deprecated 3.x APIs removed[^4]. |

## References

[^1]: Thompson, Farley, Barker, Gee, Stewart, "Disruptor: High performance alternative to bounded queues for exchanging data between concurrent threads" — 2011. https://lmax-exchange.github.io/disruptor/disruptor.html
[^2]: Martin Fowler, "The LMAX Architecture" — 2011-07-12. https://martinfowler.com/articles/lmax.html
[^3]: Disruptor User Guide (overview, wait strategies, DSL). https://lmax-exchange.github.io/disruptor/user-guide/index.html
[^4]: LMAX Disruptor 4.0 release notes / GitHub releases. https://github.com/LMAX-Exchange/disruptor/releases

## Tags

java, concurrency, lock-free, ring-buffer, low-latency, inter-thread-messaging, high-throughput, jvm, mechanical-sympathy, event-processing
