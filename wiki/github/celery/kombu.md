# celery/kombu

> The messaging abstraction layer beneath Celery — one AMQP-shaped API over RabbitMQ, Redis, SQS, and a handful of other brokers.

[GitHub repo](https://github.com/celery/kombu) ·
[Documentation](https://kombu.readthedocs.io/) ·
[License: BSD-3-Clause](https://github.com/celery/kombu/blob/main/LICENSE)

## Overview

Kombu is a Python messaging library that presents a single AMQP-style interface — producers, exchanges, queues, routing keys — and maps it onto whichever broker you actually run. It began as the successor to `carrot`, and its main job in the ecosystem is to be the transport layer that Celery sits on top of[^1]. Most people who use Kombu do so transitively through Celery; a smaller set use it directly when they want AMQP semantics without a full task framework.

The defining design choice is the split between *native* and *virtual* transports. Native transports (`py-amqp`, `qpid`) speak a real AMQP protocol to a real AMQP broker such as RabbitMQ. Virtual transports (Redis, Amazon SQS, MongoDB, ZooKeeper, in-memory, and a few others) emulate AMQP concepts — exchanges, bindings, fanout — on top of brokers that have no such primitives[^2]. This is what lets Celery advertise "works with Redis or RabbitMQ or SQS" behind one configuration knob.

That abstraction is also the source of most of Kombu's sharp edges. Exchange/queue semantics that are cheap and correct on RabbitMQ are simulated at some cost, or only partially, on the virtual transports — and the gaps (no real fanout on SQS, no server-side TTL on Redis, in-memory-only declarations on several backends) are exactly where production surprises come from.

## Getting Started

```bash
pip install kombu
# transport extras, e.g.:
pip install "kombu[redis]"    # or kombu[sqs], kombu[mongodb]
```

```python
from kombu import Connection, Exchange, Queue

media_exchange = Exchange("media", "direct", durable=True)
video_queue = Queue("video", exchange=media_exchange, routing_key="video")

def process_media(body, message):
    print(body)
    message.ack()

with Connection("amqp://guest:guest@localhost//") as conn:
    producer = conn.Producer(serializer="json")
    producer.publish(
        {"name": "/tmp/lolcat1.avi", "size": 1301013},
        exchange=media_exchange, routing_key="video",
        declare=[video_queue],          # declare so the message is deliverable
    )

    with conn.Consumer(video_queue, callbacks=[process_media]):
        conn.drain_events(timeout=1)
```

Swapping `amqp://` for `redis://` or `sqs://` changes the broker without touching the producer/consumer code — that portability is the whole selling point.

## Architecture / How It Works

The core objects are declarative and broker-agnostic: an `Exchange` or `Queue` is just a pickleable description until it is *bound* to a channel, at which point operations (`declare`, `delete`, `publish`) actually hit the broker. `Connection` lazily opens a real connection and hands out `Channel`s; `Producer` and `Consumer` wrap a channel.

Underneath, a transport driver implements the broker-specific behavior:

- **Native (`py-amqp`)** — a pure-Python AMQP 0.9.1 client (also a celery-org project). Exchanges, bindings, fanout, priority, and TTL are all handled server-side by RabbitMQ, so Kombu is a thin translation layer here.
- **Virtual transports** — a shared base class (`kombu.transport.virtual`) reimplements AMQP routing in Python. Exchanges and bindings are tracked in the transport, and messages are stored in the broker's native structures: Redis lists/pub-sub, SQS queues, MongoDB collections. Fanout, topic matching, and priority are emulated to the extent the backend allows[^2].

Two properties follow directly from that emulation. First, on several virtual transports the exchange/queue *declarations live only in memory of the connected client*, so every producer and consumer must declare the topology it uses — there is no server holding it[^2]. Second, capability is uneven: Redis supports priority (via multiple lists) and fanout (via pub/sub) but not TTL; SQS supports neither priority nor native fanout (fanout requires an SNS fan-out topic, off by default)[^2]. Kombu's own docs ship a transport comparison table precisely because the differences are load-bearing.

Reliability helpers sit on top: `Connection.ensure()` / `ensure_connection()` wrap operations in retry-with-backoff, connection and producer **pools** amortize the cost of establishing channels, and serialization/compression are pluggable (JSON is the safe default; pickle exists but is a deserialization risk).

## Production Notes

**The Redis transport has a visibility-timeout model, not true acks.** A consumed-but-unacked message is redelivered after `visibility_timeout` (default 3600s) elapses. Tasks that run longer than the timeout get redelivered while still running — a classic duplicate-execution footgun. Fanout via Redis pub/sub is fire-and-forget: a message published while no subscriber is connected is simply lost, and pub/sub traffic is not persisted.

**SQS is the most constrained backend.** No message priority, no native fanout (you opt into SNS with `supports_fanout`, which adds cost and setup), long-poll intervals affect latency and API-call billing, and you generally must use `predefined_queues` or grant broad queue-management IAM. Treat it as "durable simple queue," not "AMQP."

**Declare topology everywhere.** Because virtual transports keep declarations client-side, a consumer that assumes a publisher already created an exchange/queue will silently receive nothing. The Kombu idiom — publishers and consumers both `declare=[...]` — is not optional on these transports.

**Serialization defaults matter.** Older Celery/Kombu stacks defaulted to pickle; accepting pickled messages from an untrusted broker is remote code execution. Pin `serializer="json"` and restrict `accept` content types.

**Version coupling with Celery.** Kombu is released in lockstep-ish with Celery and pins a narrow `amqp` (py-amqp) range. Upgrading Kombu underneath a fixed Celery, or vice versa, frequently trips dependency-resolver conflicts; upgrade the pair together and read both changelogs. Async consumption (`kombu.asynchronous`) exists to serve Celery's event loop but is not a general asyncio API — Kombu is fundamentally synchronous.

## When to Use / When Not

**Use when:**
- You need broker portability — the same code against RabbitMQ in prod and Redis/in-memory in tests.
- You want AMQP producer/consumer/routing semantics without hand-rolling protocol handling.
- You are extending or integrating with Celery and need to touch the transport layer directly.

**Avoid when:**
- You target exactly one broker and want its full native feature set — use that broker's own client (`redis-py`, `pika`) instead of a lowest-common-denominator abstraction.
- You need asyncio-native messaging — Kombu is synchronous; reach for `aio-pika` or an async broker SDK.
- You want a full task queue — use Celery (or an alternative) rather than assembling one on raw Kombu.

## Alternatives

- celery/py-amqp — use directly when you only need a raw AMQP 0.9.1 client for RabbitMQ and no cross-broker abstraction.
- pika/pika — use when you want a widely adopted pure-Python RabbitMQ client with direct control over the protocol.
- mosquito/aio-pika — use when you need asyncio-native AMQP; Kombu's consumer loop is synchronous.
- Bogdanp/dramatiq — use when you want a simpler, more opinionated task-processing library rather than the Celery+Kombu stack.
- redis/redis-py — use when Redis is your only broker and transport portability buys you nothing.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010 | Created as successor to `carrot`; AMQP messaging for Python[^1]. |
| 3.x | ~2014 | Broad virtual-transport lineup (Redis, SQS, MongoDB, ZooKeeper). |
| 4.x | ~2017 | Aligned with Celery 4; py-amqp as the default AMQP transport. |
| 5.0 | 2020 | Released alongside Celery 5.0; Python 3 only[^3]. |
| 5.6.2 | 2026 | Current release line at time of writing[^3]. |

## References

[^1]: Kombu documentation — "Introduction" (Kombu as an AMQP messaging framework and Celery's transport layer; successor to carrot). https://kombu.readthedocs.io/en/latest/introduction.html
[^2]: Kombu README / docs — Transport Comparison table and virtual-transport footnotes (in-memory declarations, SQS fanout via SNS, per-transport priority/TTL support). https://github.com/celery/kombu/blob/main/README.rst
[^3]: Kombu on PyPI — release history and current version. https://pypi.org/project/kombu/#history

## Tags

python, messaging, message-queue, amqp, rabbitmq, redis, amazon-sqs, celery, broker-abstraction, task-queue, library
