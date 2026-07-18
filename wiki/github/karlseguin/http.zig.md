# karlseguin/http.zig

> A from-scratch HTTP/1.1 server library for Zig — the de facto base layer of
> the Zig web ecosystem, deliberately not built on `std.http.Server`.

[GitHub repo](https://github.com/karlseguin/http.zig) ·
[License: MIT](https://github.com/karlseguin/http.zig/blob/master/LICENSE)

## Overview

http.zig (module name `httpz`) is an HTTP/1.1 server library by Karl Seguin,
started in March 2023[^1]. It exists because `std.http.Server` is slow and
assumes well-behaved clients; most other Zig HTTP servers wrap either the std
implementation (inheriting its performance) or C libraries[^2]. httpz is pure
Zig with its own socket handling and parser; the author reports ~140K
requests/second for a basic request on an Apple M2[^2]. Higher-level Zig
frameworks — JetZig and Tokamak — build on it[^2], making httpz roughly what
`net/http` is to Go: the substrate others frame over.

The defining tension is Zig itself. Zig is pre-1.0 and breaks its standard
library every release, so httpz maintains a branch per Zig version
(`zig-0.15`, `dev`, etc.); `master` tracks the latest stable Zig[^2]. As of
mid-2026 master targets Zig 0.16 and the author explicitly labels that port
experimental and "not well tested"[^2]. Adopting httpz means adopting this
churn: upgrade pain comes from the language more than from the library.

At ~1.6k stars with a single primary maintainer, it is small by web-framework
standards but among the most-used Zig libraries, and actively developed.

## Getting Started

```bash
zig fetch --save "git+https://github.com/karlseguin/http.zig#master"
```

```zig
// build.zig
const httpz = b.dependency("httpz", .{ .target = target, .optimize = optimize });
exe.root_module.addImport("httpz", httpz.module("httpz"));
```

```zig
// Zig 0.16 / master API
const std = @import("std");
const httpz = @import("httpz");

pub fn main(init: std.process.Init) !void {
    var server = try httpz.Server(void).init(init.io, init.gpa, .{
        .address = .localhost(5882),
    }, {});

    var router = try server.router(.{});
    router.get("/api/user/:id", getUser, .{});
    try server.listen(); // blocks
}

fn getUser(req: *httpz.Request, res: *httpz.Response) !void {
    try res.json(.{ .id = req.param("id").?, .name = "Teg" }, .{});
}
```

## Architecture / How It Works

**Comptime handler generics.** The core type is `httpz.Server(H)` where `H`
is your application handler. With `H = void`, actions are plain
`fn(*Request, *Response) !void`; with a real handler, the instance is passed
to every action (the idiomatic way to share a DB pool or config). Signatures
are inferred at compile time — no interface boilerplate, no dispatch cost.

**Layered override points.** A handler can define, in increasing order of
control: `dispatch` (wrap every action — timing, auth, per-request context
structs), `notFound`, `uncaughtError`, and finally `handle`, which bypasses
httpz's router and dispatching entirely; JetZig hooks `handle` to supply its
own routing[^2] — frameworks consume httpz as a raw connection/parsing
engine, applications as a router.

**Memory model.** Each request gets `req.arena` / `res.arena` — a
configurable thread-local buffer falling back to an `std.heap.ArenaAllocator`,
freed after the response is written[^2]. Response data must stay valid until
*after* the action returns, so the arena (or `res.writer()`) is the intended
path for dynamic bodies. Request bodies land in a static per-connection
buffer, a large-buffer pool, or a dynamic allocation depending on size;
`request.lazy_read_size` switches large bodies to streaming via
`req.reader(timeout_ms)` instead of full buffering.

**Batteries at the protocol level, not above it.** Router with `:params`,
middleware, query/form/multipart/JSON body parsing, server-sent events,
internal metrics, a `httpz.testing` harness for unit-testing actions, and
WebSocket upgrades (via the author's companion websocket.zig). No templating,
ORM, or sessions — JetZig/Tokamak territory.

## Production Notes

- **No TLS.** httpz speaks plaintext HTTP/1.1 only — no HTTP/2 or HTTP/3;
  terminate TLS at a reverse proxy (nginx, Caddy).
- **Zig version pinning is your main upgrade chore.** Pin a `zig-X.YY` branch
  in `build.zig.zon`. The 0.15 cycle changed the writer API (`res.writer()`
  takes a buffer — pass `&.{}`, httpz buffers internally)[^2]; the 0.16 port
  rewires I/O through the new `std.Io` interface and is flagged experimental.
- **Form parsing is off by default.** `request.max_form_count` and
  `request.max_multiform_count` default to 0 — form/multipart parsing yields
  nothing until raised[^2]. `formData()` and `multiFormData()` are mutually
  exclusive per request; multipart unescapes names in-place, subtly altering
  a later `req.body()`.
- **Lifetime discipline.** Setting `res.body` to memory that dies when your
  action returns is the classic first bug; use `res.arena`.
- **Single-maintainer risk.** Karl Seguin is prolific and responsive (he also
  maintains pg.zig and websocket.zig), but the bus factor is 1, with no
  corporate backing. Issue volume is low (~15 open): little churn, but a
  small contributor pool.
- The 140K req/s figure is author-reported on an M2 for a trivial handler[^2]
  — an upper bound, not a capacity plan.

## When to Use / When Not

**Use when:**
- You are writing a Zig service and need an HTTP layer faster and more
  defensive than `std.http.Server`.
- You want a library, not a framework — routing plus request/response
  primitives, with your own architecture on top.
- You need WebSockets or SSE from the same server.

**Avoid when:**
- You need TLS in-process or HTTP/2+ — proxy in front, or use another stack.
- You want rails-style productivity in Zig — JetZig or Tokamak (both built on
  httpz) give you more out of the box.
- Your team won't track Zig's breaking releases; a Go or Rust HTTP stack is
  operationally far more stable.

## Alternatives

- zigzap/zap — wraps the C library facil.io; mature C core, but not pure Zig.
  Use it for a batteries-heavier microframework if the C dependency is fine.
- jetzig-framework/jetzig — full MVC-style framework built on httpz; use it
  when you want templating, generators, and conventions rather than a library.
- cztomsik/tokamak — dependency-injection-oriented server framework on httpz;
  use it for structured larger apps with DI ergonomics.
- ziglang/zig (`std.http.Server`) — zero dependencies; fine for internal
  tools and tests where throughput and hostile clients don't matter.

## History

| Era | Date | Notes |
|-----|------|-------|
| Initial | 2023-03 | Repo created; HTTP/1.1 server independent of `std.http.Server`[^1]. Branch-per-Zig-version model: `master` = latest stable Zig, `dev` = Zig master, `zig-X.YY` for older[^2]. |
| Zig 0.14 support | 2025-03 | Tracks Zig 0.14.0 release. |
| Zig 0.15 support | 2025-08 | New `std.Io.Writer` interface; `res.writer()` gains a buffer parameter[^2]. |
| Zig 0.16 port | 2026 | `main(init: std.process.Init)` / `std.Io` rework; author labels it experimental[^2]. |

## References

[^1]: GitHub repository metadata — created 2023-03-13. https://github.com/karlseguin/http.zig
[^2]: http.zig README (versions, alternatives, performance claim, handler/dispatch, memory & arenas, configuration). https://github.com/karlseguin/http.zig#readme

## Tags

zig, http-server, http-1-1, web, networking, server-library, routing, websocket, server-sent-events, low-level
