# socketio/socket.io

Socket.IO — realtime, bidirectional, event-based communication library for Node.js + browser. The classic "WebSockets with fallback + rooms + acks" library.

## What it is

A TypeScript library that wraps WebSockets (with HTTP long-polling fallback) into an event-emitter API for real-time apps. Provides rooms, namespaces, broadcasting, acknowledgements, and reconnection out of the box. The fallback layer is the historical differentiator — in environments where WebSockets are blocked, Socket.IO degrades to long-polling rather than failing.

## Key features

- WebSocket-first with HTTP long-polling fallback.
- Rooms + namespaces for organizing subscriptions.
- Broadcast, acknowledgments, message ordering.
- Auto-reconnection with exponential backoff.
- Server: Node.js (Deno + Bun support). Client: browser, Node, React Native.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Node.js on the server; browser-compatible client.

## When to reach for it

- You need real-time bidirectional communication and want the proven library.
- You need the HTTP long-polling fallback for restrictive networks.
- You want batteries-included (rooms, acks, reconnection).

## When *not* to reach for it

- You want raw WebSockets — `ws` library is lighter.
- You want server-sent-events — different protocol, lighter for one-way streams.
- You're shipping at scale where Socket.IO's overhead matters — consider raw WebSockets.

## Maturity signal

Actively maintained, 15+ years. MIT clean.

## Alternatives

- Raw WebSockets via `ws`.
- SignalR (.NET).
- Phoenix Channels (Elixir).

## Tags

typescript, nodejs, websocket, real-time, library, mit-license
