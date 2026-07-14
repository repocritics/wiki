# websockets/ws

> The de facto WebSocket implementation for Node.js — a lean RFC 6455 client and server with no runtime dependencies.

[GitHub repo](https://github.com/websockets/ws) ·
[npm package](https://www.npmjs.com/package/ws) ·
[License: MIT](https://github.com/websockets/ws/blob/master/LICENSE)

## Overview

ws is a WebSocket client and server library for Node.js, in continuous
development since 2011[^1]. It implements RFC 6455 (the WebSocket protocol) and
the permessage-deflate extension (RFC 7692), and passes the Autobahn conformance
test suite for both roles[^2]. It is one of the most-depended-on packages in the
Node ecosystem: Socket.IO's transport layer (engine.io), most GraphQL
subscription servers, and a large share of Node real-time tooling sit on top of
it, so its download volume vastly exceeds its GitHub star count.

The defining characteristic is scope discipline. ws does the WebSocket framing
protocol and nothing else — no reconnection, no rooms, no pub/sub, no
heartbeat-driven liveness detection, no browser build. It is a protocol
implementation, not a framework. That is a deliberate tradeoff: the surface area
is small and the code is auditable, but you inherit responsibility for the
operational concerns (liveness, backpressure, broadcast fan-out, auth) that
higher-level libraries fold in. Teams that want batteries included generally
reach for Socket.IO; teams that want the raw protocol reach here.

Note that ws does **not** run in the browser[^3]. The "client" role in its docs
means a Node process acting as the connecting party; browser code must use the
native `WebSocket` object.

## Getting Started

```bash
npm install ws
```

```js
// server.js — echo server
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', function (ws) {
  ws.on('error', console.error);          // REQUIRED — see Production Notes
  ws.on('message', function (data, isBinary) {
    ws.send(data, { binary: isBinary });
  });
});
```

```js
// client.js — a Node process acting as WebSocket client
import WebSocket from 'ws';

const ws = new WebSocket('ws://localhost:8080');
ws.on('error', console.error);
ws.on('open', () => ws.send('hello'));
ws.on('message', (data) => console.log('received: %s', data));
```

## Architecture / How It Works

ws parses the WebSocket wire protocol in JavaScript, layered directly on Node's
`net`/`tls`/`http` primitives. The core is two classes: a **Receiver** (a state
machine that reassembles inbound frames into messages, validating opcodes,
masking, fragmentation, and UTF-8) and a **Sender** (which frames, masks, and
optionally compresses outbound data). A `WebSocket` instance wraps one TCP/TLS
socket plus one Receiver/Sender pair.

The server side is intentionally thin. `WebSocketServer` does not own a network
listener by default in the advanced path — it hooks the HTTP `upgrade` event,
performs the RFC 6455 handshake (validating `Sec-WebSocket-Key`, echoing the
computed `Sec-WebSocket-Accept`), and hands back a `WebSocket`. The `noServer:
true` mode exposes `handleUpgrade()` so you can route multiple WebSocket servers
onto one HTTP server, or run your own authentication before the upgrade
completes. This composability is the reason ws integrates cleanly with Express,
Fastify, and bare `http.createServer`.

Two optional native addons accelerate hot paths: **bufferutil** (frame
masking/unmasking) and **utf-8-validate** (text-frame validation)[^4]. Both are
optional dependencies — ws falls back to pure-JS implementations if they are
absent or fail to compile, so installs never hard-fail on a missing C++ toolchain.
On Node 18.14.0+ the built-in `buffer.isUtf8()` supersedes utf-8-validate. Both
can be force-disabled via `WS_NO_BUFFER_UTIL` / `WS_NO_UTF_8_VALIDATE`, which the
docs recommend as supply-chain hardening in shared environments.
`createWebSocketStream()` additionally adapts a connection into a Node Duplex
stream for `pipe()`-based code.

## Production Notes

**Unhandled `'error'` events crash the process.** Since 8.0.0, a `WebSocket` (and
each server-side connection) that emits `'error'` with no listener follows Node's
EventEmitter rule and throws, taking down the process. Every connection —
client and server-side — needs its own `ws.on('error', …)`. This is the single
most common ws production incident and the reason every README example includes
the listener.

**No built-in liveness detection.** A dropped connection (cable pulled, NAT
timeout) leaves both ends unaware. ws does not auto-detect this; you must run a
ping/pong heartbeat yourself — the canonical pattern is a `setInterval` that
`ws.ping()`s every client and `terminate()`s those that did not `pong` since the
last tick. Pong replies to your pings are automatic, but the detection loop is
your code.

**No auto-reconnect, manual backpressure.** The client does not reconnect on
drop — wrap ws yourself or use a library. `ws.send()` buffers if the socket is
not draining, and `ws.bufferedAmount` is the only signal; high-throughput senders
(especially broadcast fan-out over `wss.clients`) must check it or risk unbounded
memory growth on slow consumers.

**permessage-deflate is a memory footgun.** Compression is disabled by default on
the server and enabled by default on the client. The maintainers explicitly warn
against enabling it server-side without load-testing: Node's zlib, under
concurrency on Linux, can hit catastrophic memory fragmentation[^5]. Also cap
`maxPayload` (default 100 MiB) — compressed frames can decompress into far larger
buffers, a classic decompression-bomb DoS vector.

**Security advisories.** CVE-2024-37890 (fixed in 8.17.1, backported to 7.5.10 /
6.2.3 / 5.2.4) was a DoS: a client sending a request with a very large number of
HTTP headers could crash the server; the mitigation caps headers via the
`maxHeadersCount`/`headersTimeout` path[^6]. Pin to patched versions and keep the
optional addons current, since they are native code.

**Runtime overlap.** Node 22+, Bun, and Deno now ship a native browser-compatible
`WebSocket` *client*, reducing the need for ws on the client side — but none ship
a WebSocket *server*, which remains ws's strongest reason to exist.

## When to Use / When Not

**Use when:**
- You need a WebSocket **server** in Node and want the reference-grade,
  low-overhead implementation.
- You want direct control of the handshake (auth before upgrade, multiple servers
  on one port, custom routing).
- You want minimal dependencies and an auditable protocol layer.

**Avoid when:**
- You want rooms, reconnection, acknowledgements, and transport fallback out of
  the box — use Socket.IO.
- You need maximal throughput/connection density — a C++-backed server will beat
  a JS frame parser.
- You are writing browser client code — use the native `WebSocket`.

## Alternatives

- socketio/socket.io — higher-level real-time framework (rooms, reconnection, acks); use it when you want batteries included rather than the raw protocol.
- uNetworking/uWebSockets.js — C++-backed WebSocket/HTTP server; use it when connection density and throughput dominate and you can accept a different API.
- theturtle32/WebSocket-Node — older pure-JS implementation (the `websocket` npm package); use it only for legacy code already on it.
- primus/primus — abstraction over multiple real-time transports; use it when you want to swap engines (ws, sockjs, engine.io) behind one API.
- Deno / Bun native WebSocket — use when your runtime already ships a first-class server and you want zero dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-11 | First release; project created on GitHub[^1]. |
| 6.0 | 2018-09 | `Sender`/`Receiver` refactor; API cleanup. |
| 7.0 | 2019-06 | Dropped end-of-life Node versions; stricter close semantics. |
| 8.0 | 2021-07 | `WebSocketServer` named export; **unhandled `'error'` now throws**; Node 10+ required. |
| 8.17.1 | 2024-06 | Fix for CVE-2024-37890 (header-flood DoS)[^6]. |

Dates for the 6.x/7.x/8.x majors are approximate to the month; consult the
GitHub releases page[^7] for exact changelog entries.

## References

[^1]: websockets/ws repository, created 2011-11-09. https://github.com/websockets/ws
[^2]: Autobahn conformance reports (server / client). http://websockets.github.io/ws/autobahn/servers/ · http://websockets.github.io/ws/autobahn/clients/
[^3]: ws README, "This module does not work in the browser." https://github.com/websockets/ws#ws-a-nodejs-websocket-library
[^4]: bufferutil and utf-8-validate optional native addons. https://github.com/websockets/bufferutil · https://github.com/websockets/utf-8-validate
[^5]: ws README, WebSocket compression / Node zlib memory-fragmentation warning; nodejs/node#8871. https://github.com/nodejs/node/issues/8871
[^6]: CVE-2024-37890 — ws DoS via excessive HTTP headers; fixed in 8.17.1. https://github.com/advisories/GHSA-3h5v-q93c-6h6q
[^7]: ws releases (changelog). https://github.com/websockets/ws/releases

## Tags

javascript, nodejs, websocket, rfc-6455, real-time, networking, server, client, protocol, library
