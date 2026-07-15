# netty/netty

> Event-driven asynchronous network framework for the JVM — the transport layer under most Java infrastructure you've heard of.

[GitHub repo](https://github.com/netty/netty) ·
[Official website](https://netty.io) ·
[License: Apache-2.0](https://github.com/netty/netty/blob/4.2/LICENSE.txt)

## Overview

Netty is a low-level networking framework for building protocol servers and
clients on the JVM. It began at JBoss (Trustin Lee) and has been a standalone
project since 2010[^1]. It is not an application framework or a web server; it
is the plumbing — non-blocking I/O, buffer management, and a pipeline of
composable handlers — that higher-level systems build on. gRPC-Java, Cassandra,
Elasticsearch, Spark, Vert.x, Akka, Play, Hadoop RPC, and countless proprietary
services all sit on top of it[^2]. If you write directly against Netty you are
usually implementing a custom wire protocol or squeezing latency out of an
existing one.

The framework's defining tension is control versus footguns. Netty exposes the
event loop, the byte buffers, and the reference-counting lifecycle directly, so
you get near-hand-tuned performance and can implement any protocol. The cost is
that mistakes the JVM normally hides — blocking the I/O thread, leaking a
manually reference-counted buffer, mutating a handler shared across channels —
become production incidents rather than compile errors. Netty rewards operators
who understand its threading and memory model and punishes those who treat it
as a black box.

A second, quieter tension in 2026 is Project Loom. Virtual threads make
blocking, thread-per-connection code cheap again, which removes the original
reason many teams reached for an async framework at all. Netty remains the
default for high-throughput protocol work, but for ordinary request/response
services the async-vs-blocking calculus has shifted.

## Getting Started

Netty is a Maven/Gradle dependency; there is no CLI. The `netty-all` artifact
pulls in everything, or depend on the specific modules you need. Netty 4.2
requires Java 8 or newer; the optional `io_uring` native transport requires
Java 9+[^3].

```xml
<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-all</artifactId>
  <version>4.2.0.Final</version>
</dependency>
```

```java
// A minimal echo server. Two EventLoopGroups: one accepts, one does I/O.
EventLoopGroup boss = new NioEventLoopGroup(1);
EventLoopGroup work = new NioEventLoopGroup();
try {
  ServerBootstrap b = new ServerBootstrap();
  b.group(boss, work)
   .channel(NioServerSocketChannel.class)
   .childHandler(new ChannelInitializer<SocketChannel>() {
     protected void initChannel(SocketChannel ch) {
       ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
         public void channelRead(ChannelHandlerContext ctx, Object msg) {
           ctx.writeAndFlush(msg);   // echo the ByteBuf straight back
         }
       });
     }
   });
  b.bind(8080).sync().channel().closeFuture().sync();
} finally {
  boss.shutdownGracefully();
  work.shutdownGracefully();
}
```

## Architecture / How It Works

The core objects are `Channel`, `EventLoop`, `ChannelPipeline`, and `ByteBuf`.

- **EventLoopGroup / EventLoop** — a small pool of threads, each owning a
  selector and a set of channels. Every I/O event and every handler callback for
  a given channel runs on that channel's single event loop thread. This is the
  most important fact about Netty: a channel is effectively single-threaded, so
  handlers need no locking *as long as you never block the loop*.
- **ChannelPipeline** — an ordered chain of `ChannelHandler`s. Inbound events
  (reads) flow head-to-tail; outbound events (writes) flow tail-to-head. Codecs
  (`ByteToMessageDecoder`, `MessageToByteEncoder`) are just handlers, so a
  protocol is assembled by stacking decode/encode/business handlers.
- **ByteBuf** — Netty's replacement for `java.nio.ByteBuffer`: separate read and
  write indices, pooling, and composite (zero-copy gather) buffers. Pooled
  direct buffers are reference-counted and must be explicitly released.
- **Transports** — the same API runs over multiple backends: portable `nio`, and
  native `epoll` (Linux), `kqueue` (BSD/macOS), and `io_uring` (Linux). The
  native transports reduce GC pressure and expose OS-specific socket options;
  they are drop-in swaps at bootstrap time but ship as separate,
  platform-classified artifacts.

The coupling that matters: the threading model, the pipeline, and manual buffer
lifecycle are one system. You cannot adopt the ergonomics without adopting the
discipline. Almost every Netty production bug traces back to violating one of
three invariants — don't block the event loop, don't leak a buffer, don't share
a stateful handler across channels without `@Sharable` and thread safety.

## Production Notes

- **Direct memory, not heap, is the constraint.** Pooled buffers live in
  off-heap direct memory governed by `-XX:MaxDirectMemorySize` and Netty's own
  `io.netty.maxDirectMemory`, not by `-Xmx`. `OutOfDirectMemoryError` is a
  distinct failure mode from a normal heap OOM and needs its own alerting.
- **Buffer leaks are silent by default.** A `ByteBuf` you forget to `release()`
  is reclaimed only when the JVM GCs the object, long after the pool is
  exhausted. Netty's leak detector samples ~1% of allocations at default level;
  run `-Dio.netty.leakDetection.level=paranoid` in staging to catch leaks with a
  full allocation stack trace before they reach production.
- **Never block the I/O thread.** A JDBC call, a `synchronized` section, or a
  DNS lookup inside a handler stalls every other connection on that event loop.
  Offload blocking work to a separate `EventExecutorGroup` added at pipeline
  registration.
- **Backpressure is manual.** If a peer reads slower than you write, the outbound
  buffer grows unbounded unless you check `Channel.isWritable()` and honor the
  high/low water marks. This is a common source of memory exhaustion under load.
- **Native transports are worth it but add packaging complexity.** `epoll`
  outperforms `nio` under high connection counts, but the artifact is
  OS/architecture-classified — build and CI matrices must produce the right
  classifier, and container base images must match.
- **TLS uses `SslContext`.** Netty can use the JDK provider or OpenSSL via
  `netty-tcnat` (BoringSSL); the native path is meaningfully faster but is
  another platform-specific native dependency to manage.
- **API stability is excellent.** 4.1 held source compatibility for roughly a
  decade; upgrades within a major line are usually a version bump. The abandoned
  5.0 line (below) is the reason there is no forced rewrite.

## When to Use / When Not

**Use when:**
- You are implementing a custom or binary wire protocol and need full control.
- You need to sustain very high connection counts or low tail latency.
- You are building infrastructure (a proxy, broker, database, RPC layer) rather
  than an application.

**Avoid when:**
- You just need an HTTP API — use Spring Boot, Quarkus, Micronaut, or Javalin;
  they wrap Netty (or an equivalent) and hide the sharp edges.
- Your team is not prepared to reason about event loops and off-heap memory; the
  learning curve is steep and the failure modes are subtle.
- Loom-era blocking code on virtual threads meets your throughput needs — it is
  far simpler for ordinary request/response services.

## Alternatives

- eclipse-vertx/vert.x — built on Netty; use it when you want a higher-level reactive toolkit instead of raw pipeline plumbing.
- reactor/reactor-netty — use it when you live in Reactor/Spring WebFlux and want Reactive Streams semantics over Netty.
- grpc/grpc-java — use it when your protocol is gRPC and you don't want to hand-write codecs (it uses Netty internally).
- apache/mina — the older JVM async network framework; largely superseded, relevant mainly for legacy maintenance.
- tokio-rs/tokio — use it when you're leaving the JVM for Rust and want the equivalent async I/O foundation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 3.x | 2011–2013 | Original public line; single-artifact, pre-modular API. |
| 4.0 | 2013-07 | Threading model overhaul, pooled `ByteBuf`, module split. |
| 4.1 | 2016-07 | Long-lived stable line; HTTP/2, native transports matured[^1]. |
| 5.0 | (abandoned) | Branched then cancelled ~2015; changes folded back into 4.x[^4]. |
| 4.2 | 2025 | Current development line; `io_uring` transport, Java 8 baseline[^3]. |

## References

[^1]: Netty project site and news. https://netty.io/
[^2]: Netty adopters / "who uses Netty" listing. https://netty.io/
[^3]: Netty README, "System requirements" — Netty 4.2 requires Java 8+, `io_uring` requires Java 9+. https://github.com/netty/netty/blob/4.2/README.md
[^4]: The 5.0 line was branched and later abandoned, with work merged back into 4.x; see Netty project news. https://netty.io/news/

## Tags

java, jvm, networking, async-io, event-driven, non-blocking, netty, tcp, protocol-framework, high-performance, infrastructure
