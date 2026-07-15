# OpenHFT/Chronicle-Queue

> A broker-less, off-heap, memory-mapped persisted queue for microsecond-latency Java messaging.

[GitHub repo](https://github.com/OpenHFT/Chronicle-Queue) ·
[Official website](https://chronicle.software/products/chronicle-queue/) ·
[License: Apache-2.0](https://github.com/OpenHFT/Chronicle-Queue/blob/ea/LICENSE.adoc)

## Overview

Chronicle Queue is a Java library for persisted, low-latency messaging, developed since 2013 by Peter Lawrey, Rob Austin, and the Chronicle Software team[^1]. It is not a server or a broker: there is no daemon process, no network protocol on the open-source path, and no separate storage engine. A queue is a directory of memory-mapped files on the local disk, and producers ("appenders") and consumers ("tailers") coordinate purely through those files and the operating system's page cache. The design target is single-digit-microsecond IPC between JVMs on the same host, with every message durably written to disk[^2].

The defining tradeoff is **producer-centric, off-heap, local-only**. Chronicle Queue never pushes back on the writer — it assumes disk is cheap and keeps everything, so it behaves as a large durable buffer between a fast upstream (market data, order flow, compliance events) and a slower consumer. It stores messages in off-heap, memory-mapped files, so multi-terabyte queues impose almost no on-heap footprint and generate essentially no garbage collection pressure. The cost of these choices is a narrow deployment envelope: replication, encryption, async writes, and timezone-aware rollover are Enterprise-Edition-only[^3], and the open-source library is fundamentally a same-host tool.

It is most used in finance-adjacent systems — trading, market-data capture, compliance recording, and low-latency microservice pipelines — where retaining a total-ordered, replayable log of every event with microsecond overhead matters more than convenience.

## Getting Started

Add the dependency from Maven Central (`net.openhft:chronicle-queue`); a BOM (`chronicle-bom`) is recommended to align the OpenHFT stack versions.

```xml
<dependency>
  <groupId>net.openhft</groupId>
  <artifactId>chronicle-queue</artifactId>
  <version><!-- latest release, see GitHub Releases --></version>
</dependency>
```

```java
import net.openhft.chronicle.queue.*;
import net.openhft.chronicle.wire.DocumentContext;

try (ChronicleQueue queue = ChronicleQueue.singleBuilder("/tmp/my-queue").build()) {
    // Append
    ExcerptAppender appender = queue.acquireAppender();
    try (DocumentContext dc = appender.writingDocument()) {
        dc.wire().write("event").text("hello");
    }

    // Tail (read from the start)
    ExcerptTailer tailer = queue.createTailer();
    try (DocumentContext dc = tailer.readingDocument()) {
        if (dc.isPresent()) {
            System.out.println(dc.wire().read("event").text());
        }
    }
}
```

Higher-level typed messaging uses `methodWriter` / `methodReader`, which generate proxies that serialize interface calls as messages and dispatch them on read — the idiomatic pattern for event-sourced services.

## Architecture / How It Works

A queue directory holds one **cycle file** per roll period (`.cq4`) plus a shared **table store** (`metadata.cq4t`). The roll cycle — hourly, daily, and others — is fixed by the first queue instance created against a path and is encoded in the metadata; opening the same path later with a different cycle fails, which is a common configuration footgun.

Messages are written through **Chronicle Wire**, a self-describing serialization layer that can render the same bytes as binary (for speed) or YAML-like text (for debugging and dumps). Because the format is self-describing, a reader does not need to share the writer's configuration to decode a queue — a problem that the pre-v4 "Vanilla/Indexed" formats had. Under Wire sits **Chronicle Bytes**, which wraps memory-mapped regions and, on supported JDKs, uses `sun.misc.Unsafe` for direct off-heap access. Chronicle Queue therefore pulls in the wider OpenHFT stack (Chronicle-Core, Chronicle-Bytes, Chronicle-Wire, Chronicle-Threads) and shares their JVM-version sensitivities.

Concurrency model:
- **Multiple appenders** on one host serialize through a write lock; since v5 that lock lives in the table store rather than in each cycle file.
- **Multiple tailers** read lock-free and independently; each tailer holds its own index cursor. Tailers became read-only in v5, which dropped v4's "lazy indexing" (where a tailer could write indexes) in exchange for simpler, faster internals.
- **Total ordering** is guaranteed per queue: every reader sees the same sequence, and any position is addressable by its index for replay-from-any-point semantics.

There is no broker process doing work. Durability comes from the OS: writes land in the page cache and are flushed by the kernel, so if the JVM crashes the data already written survives (the OS keeps running). This is also why Chronicle Queue depends on real memory-mapping primitives and refuses to run on network filesystems.

## Production Notes

**JDK module system.** On Java 11+ the off-heap/`Unsafe` access requires JVM flags — typically `--add-exports java.base/jdk.internal.ref=ALL-UNNAMED`, `--add-opens java.base/java.lang=ALL-UNNAMED`, and related `--add-exports`/`--add-opens` entries. Missing flags surface as `InaccessibleObjectException` or reflection errors at startup, not at build time. Each new major JDK tends to require checking the OpenHFT compatibility notes.

**No network filesystems — ever.** NFS, AFS, SAN-mounted, and similar filesystems do not provide the memory-mapping and locking primitives Chronicle Queue relies on; running on them causes silent corruption or lock failures. Sharing a queue across hosts is only supported via Chronicle Queue Replication, which is an Enterprise feature. The open-source library is single-host.

**Disk and cycle management.** Queues grow without bound by design. You reclaim space by registering a `StoreFileListener` to learn when a cycle file is released, then compress/move/delete whole cycles. There is no automatic retention/TTL in the core library. Plan capacity around "keep everything" and manage rollover explicitly.

**Latency outliers.** The steady-state 99% latency is dominated by the OS and disk subsystem. The main source of tail-latency spikes is page faults when the appender crosses into a newly mapped region; the Enterprise **pre-toucher** exists specifically to fault pages in ahead of the writer. Without it, expect occasional millisecond outliers at roll boundaries even when the median is sub-microsecond.

**Interrupts.** For performance, the library removed interrupt checking from its IO paths. Running it on threads that receive `Thread.interrupt()` can, on Windows, close the underlying `FileChannel` mid-operation. The maintainers' guidance is to avoid interrupts, or give each interrupting thread its own queue instance.

**Internal packages.** Classes under `internal`, `impl`, and `main` packages are explicitly not public API and can change in any release; only the documented `net.openhft.chronicle.queue` surface is stable. Enterprise features (replication, encryption, async mode, timezone rollover) require a commercial license and are not in this repository.

## When to Use / When Not

**Use when:**
- You need durable, total-ordered, replayable messaging with microsecond overhead on a single host (trading, market-data capture, compliance/audit logs).
- You want a GC-free buffer that absorbs bursty, non-backpressuring producers and retains days-to-years of data.
- You are event-sourcing and want deterministic replay of historical event streams at accelerated speed.

**Avoid when:**
- You need cross-host distribution, fan-out to many subscribers, or cloud/managed messaging — that requires the Enterprise edition or a different broker.
- Your storage is a network filesystem or container volume without real mmap support.
- You want a polyglot, language-neutral broker with an operational ecosystem (dashboards, consumer groups, ACLs) out of the box.
- Your throughput/latency needs are modest — the operational constraints (JVM flags, disk management, interrupt rules) are not worth it over a conventional queue.

## Alternatives

- LMAX-Exchange/disruptor — in-JVM ring buffer for ultra-low-latency inter-thread handoff; use it when you need speed but not disk persistence or cross-process IPC.
- apache/kafka — use it when you need a distributed, multi-consumer, cross-host log with a mature operational ecosystem and can accept millisecond-class latency.
- real-logic/aeron — use it when the priority is low-latency reliable UDP/IPC transport across hosts, with optional archive for persistence.
- apache/activemq (or Artemis) — use it when you want a general-purpose JMS broker with standard protocols rather than a raw persisted log.
- redis/redis Streams — use it when you already run Redis and want a simpler persisted stream with cross-host access at higher latency.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2013 | Repository created; early "Vanilla/Indexed" Chronicle with byte-oriented, non-self-describing storage[^1]. |
| v3 | ~2015 | Byte-centric API; readers needed the writer's configuration; file-per-thread limitations. |
| v4 | ~2016 | Complete rewrite around Chronicle Wire and self-describing messages; single-file appending; built-in indexing. |
| v5 | ~2017–2018 | Read-only tailers (lazy indexing dropped); write lock moved to `metadata.cq4t` table store; can read v4 queues with caveats. |
| 5.x (`ea`) | ongoing | Active early-access line on the `ea` branch; released to Maven Central as `net.openhft:chronicle-queue`[^2]. |

The project remains actively maintained, with commits into 2026 and an Enterprise edition funding ongoing development.

## References

[^1]: OpenHFT/Chronicle-Queue README, authored by Peter Lawrey and Rob Austin. https://github.com/OpenHFT/Chronicle-Queue
[^2]: Chronicle Software, "Chronicle Queue" product page and documentation. https://chronicle.software/products/chronicle-queue/
[^3]: Chronicle Queue Enterprise Edition (replication, encryption, async mode, pre-toucher, timezone rollover). https://chronicle.software/products/queue-enterprise/

## Tags

java, jvm, low-latency, messaging, persistence, off-heap, memory-mapped, queue, event-sourcing, ipc, high-frequency-trading
