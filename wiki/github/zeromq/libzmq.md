# zeromq/libzmq

> Brokerless messaging as a socket API — asynchronous message queues over TCP, IPC, and in-process transports, implementing the ZMTP wire protocol.

[GitHub repo](https://github.com/zeromq/libzmq) ·
[Official website](https://www.zeromq.org) ·
[License: MPL-2.0](https://github.com/zeromq/libzmq/blob/master/LICENSE)

## Overview

libzmq is the core C++ engine of ZeroMQ (ØMQ), a messaging library that looks
like BSD sockets but behaves like middleware: sockets carry discrete messages
(not byte streams), automatically reconnect, queue asynchronously in background
I/O threads, and fan out over multiple transports (`tcp://`, `ipc://`,
`inproc://`, multicast) behind a single API. It began at iMatix under Pieter
Hintjens and Martin Sústrik in the late 2000s and speaks ZMTP/3.1, an openly
specified wire protocol[^1] with independent reimplementations (JeroMQ in Java,
NetMQ in C#). The library is deliberately brokerless — there is no server to
deploy; topology is whatever your processes connect to each other.

The defining tradeoff: ZeroMQ gives you messaging *patterns* (REQ/REP, PUB/SUB,
PUSH/PULL, DEALER/ROUTER) instead of messaging *guarantees*. There is no
persistence, no delivery acknowledgment, no replay. When a PUB socket's queue
fills, messages are silently dropped by design. Reliability, if you need it,
is built by you on top — the ZeroMQ Guide[^2] spends most of its pages on
exactly these recipes. This makes libzmq closer to "a concurrency framework
with a network API" than to Kafka or RabbitMQ, and comparing them is a category
error both directions.

The project is mature rather than fast-moving: ~10.9k stars, API stable for a
decade, commits still landing (last push July 2026) but release cadence is slow
— 4.3.5 (October 2023) is the latest tagged release, notable mainly for
completing the relicense from LGPL-3.0-with-exception to MPL-2.0[^3].

## Getting Started

```bash
# Debian/Ubuntu
sudo apt install libzmq3-dev
# macOS
brew install zeromq
```

```c
// rep_server.c — minimal REP (reply) socket. Build: cc rep_server.c -lzmq
#include <zmq.h>
#include <stdio.h>

int main(void) {
    void *ctx = zmq_ctx_new();
    void *rep = zmq_socket(ctx, ZMQ_REP);
    zmq_bind(rep, "tcp://*:5555");

    char buf[256];
    int n = zmq_recv(rep, buf, sizeof buf - 1, 0);   // blocks for a request
    if (n >= 0) { buf[n] = '\0'; printf("received: %s\n", buf); }
    zmq_send(rep, "world", 5, 0);

    zmq_close(rep);
    zmq_ctx_term(ctx);
    return 0;
}
```

Most applications use a binding rather than the raw C API: `pyzmq` (Python),
`cppzmq` (header-only C++), CZMQ (higher-level C). The C API is the stable ABI
all of them wrap.

## Architecture / How It Works

A `zmq_ctx` owns a pool of background **I/O threads** (default 1). Application
threads talk to I/O threads through lock-free queues (the `ypipe` structure),
so `zmq_send` usually just enqueues and returns — actual socket writes,
reconnects, and handshakes happen asynchronously. Each I/O thread runs a
poller chosen at build time (epoll on Linux, kqueue on BSD/macOS, select as
fallback).

**Sockets are not thread-safe.** A classic ZeroMQ socket must be used from one
thread at a time; sharing one across threads without a full memory barrier is
undefined behavior. Thread-safe socket types (CLIENT/SERVER, RADIO/DISH) exist
but only in the DRAFT API, which is compiled out of distro packages by default
and carries no compatibility promise.

**Messages** (`zmq_msg_t`) use a small-message optimization — short payloads
are stored inline with no allocation, larger ones are reference-counted
buffers, enabling zero-copy sends via `zmq_msg_init_data`. Multipart messages
are delivered atomically: all frames or none.

**Patterns are state machines.** REQ enforces strict send/recv alternation;
ROUTER prepends an identity frame for addressing; XPUB/XSUB expose
subscription messages so you can build proxies. Subscription filtering for
PUB/SUB happens on the publisher side (since the 3.x series), saving bandwidth
but costing publisher CPU with many subscribers.

**Security** is pluggable via ZAP (ZeroMQ Authentication Protocol): NULL,
PLAIN, CURVE (CurveZMQ, elliptic-curve encryption via libsodium/tweetnacl),
and GSSAPI mechanisms[^1]. CURVE gives authenticated encryption without TLS,
but key distribution is entirely your problem.

The codebase is C++98 with optional C++11 fragments, built with autotools or
CMake. Sústrik later argued the C++ implementation itself was a mistake for a
library of this kind[^4] and founded nanomsg — worth reading as an honest
internal critique of the design.

## Production Notes

- **Silent drops at the high-water mark.** Every socket has an HWM (default
  1000 messages). PUB drops silently when a subscriber is slow; PUSH blocks.
  Most "ZeroMQ lost my messages" reports are HWM behavior working as
  documented. Monitor with `zmq_socket_monitor` and size HWM deliberately.
- **The slow-joiner problem.** A SUB socket connected "just before" a PUB
  sends will miss the first messages — the TCP handshake and subscription
  propagation race the publisher. Synchronize startup explicitly (the Guide's
  "node coordination" pattern) or accept the loss.
- **REQ/REP deadlocks.** The strict lockstep state machine means one lost
  reply wedges a REQ socket permanently. Real systems use DEALER/ROUTER with
  timeouts and retries (Lazy Pirate and friends[^2]); bare REQ/REP is a demo
  pattern.
- **`zmq_ctx_term` hangs.** Default `ZMQ_LINGER` is -1 (infinite): context
  termination blocks until all queued messages are sent. Set LINGER to 0 or a
  bounded value on every socket before shutdown, or your process will hang on
  exit with a dead peer.
- **Reconnect hides failures.** TCP sockets reconnect transparently, which is
  convenient until you realize messages queued during the outage may be gone
  and no error was ever surfaced. There is no delivery notification API;
  heartbeat at the application level (or via ZMTP heartbeats, 4.2+).
- **Security patch history.** The 4.3.x line shipped fixes for remotely
  triggerable memory bugs, e.g. CVE-2019-6250 (fixed in 4.3.1)[^5]. Given the
  slow release cadence, track the git master or distro backports for CVE
  fixes rather than waiting for tags.
- **DRAFT API trap.** UDP, RADIO/DISH, CLIENT/SERVER, and WebSocket transport
  live behind `--enable-drafts`. Prebuilt distro packages exclude them;
  depending on drafts means building libzmq yourself forever.

## When to Use / When Not

**Use when:**
- You need low-latency process-to-process or in-process messaging without
  deploying and operating a broker.
- Your topology fits the built-in patterns (pipelines, fan-out, request
  distribution) and you can tolerate or handle message loss.
- You need one messaging API across threads (`inproc://`), processes
  (`ipc://`), and machines (`tcp://`).
- You are building your own protocol/middleware and want framing, queuing,
  and reconnection solved beneath you.

**Avoid when:**
- You need durable, acknowledged, replayable delivery — that is Kafka/
  RabbitMQ/NATS JetStream territory; building it on ZeroMQ means rewriting
  half a broker yourself.
- You need request/response with schemas, deadlines, and load balancing out
  of the box — gRPC is the mainstream answer.
- Your team expects operational visibility (queues you can inspect, dead
  letter handling) — brokerless means the state lives in opaque per-socket
  buffers.
- You want a memory-safe dependency surface: this is a large C++98 codebase
  parsing untrusted network input, with a real CVE history.

## Alternatives

- nanomsg/nng — Sústrik-lineage successor, C, cleaner scalability protocols;
  use when you want ZeroMQ-style patterns with a simpler, thread-safe API.
- nats-io/nats-server — use when you want lightweight pub/sub but with a
  broker, subject-based routing, and optional persistence (JetStream).
- rabbitmq/rabbitmq-server — use when you need acknowledged, routed, durable
  queues and operator tooling more than raw latency.
- apache/kafka — use when the problem is a replayable event log at scale,
  not point-to-point messaging.
- grpc/grpc — use when you need typed RPC with deadlines, streaming, and
  ecosystem-wide interop rather than raw messaging patterns.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.x | 2010–2011 | Early stable series; ZeroMQ gains adoption as "sockets on steroids". |
| 3.2 | 2012 | Wire protocol rework; publisher-side filtering. Sústrik departs, forks Crossroads I/O, later starts nanomsg[^4]. |
| 4.0 | 2013 | CURVE security, ZAP authentication, ZMTP/3.0[^1]. |
| 4.1 | 2015-02 | ZMTP/3.1, socket monitoring improvements. |
| 4.2 | 2016-12 | ZMTP heartbeats; `inproc://` connect-before-bind. |
| 4.3 | 2019-01 | Ongoing 4.3.x series; multiple security fixes (CVE-2019-6250 in 4.3.1)[^5]. |
| 4.3.5 | 2023-10 | Latest release; relicense to MPL-2.0 completed[^3]. |

## References

[^1]: ZMTP — ZeroMQ Message Transport Protocol specifications. https://rfc.zeromq.org/spec/23/
[^2]: Pieter Hintjens, "ØMQ - The Guide" — the canonical patterns/reliability text. https://zguide.zeromq.org/
[^3]: libzmq 4.3.5 release notes (MPL-2.0 relicense) — 2023-10. https://github.com/zeromq/libzmq/releases/tag/v4.3.5
[^4]: Martin Sústrik, "Why should I have written ZeroMQ in C, not C++". https://250bpm.com/blog:4/
[^5]: CVE-2019-6250 — pointer arithmetic vulnerability in ZMTP v2.0 handling, fixed in 4.3.1. https://nvd.nist.gov/vuln/detail/CVE-2019-6250

## Tags

c++, messaging, networking, pubsub, message-queue, brokerless, sockets, zmtp, concurrency, low-latency
