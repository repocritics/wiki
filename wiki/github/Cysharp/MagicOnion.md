# Cysharp/MagicOnion

> Unified realtime and API RPC framework for .NET and Unity, using C# interfaces as the schema instead of `.proto` files.

[GitHub repo](https://github.com/Cysharp/MagicOnion) ·
[Official website](https://cysharp.github.io/MagicOnion/) ·
[License: MIT](https://github.com/Cysharp/MagicOnion/blob/main/LICENSE)

## Overview

MagicOnion is an RPC framework built on top of gRPC by Cysharp, the Japanese studio behind UniTask, MessagePack-CSharp, and MemoryPack[^1]. Its defining decision is to treat plain C# interfaces as the service contract — the same interface is referenced by both server and client projects — so there is no `.proto` file, no protoc step, and no generated DTO layer. Method arguments and return values are serialized with MessagePack rather than Protocol Buffers. This makes it fast to build a service if both ends are C#, and effectively unusable if either end is not.

The framework offers two distinct service shapes. **Unary** services look like Web API calls: a method returns `UnaryResult<T>` and maps onto a single gRPC unary call. **StreamingHub** is a persistent bidirectional connection over a gRPC duplex stream, modeled after SignalR/Socket.io hubs — the server can push messages to individual clients or broadcast to groups. The two can be mixed in one server. The primary audience is teams building a .NET backend for a Unity game client, though it is equally usable for pure server-to-server or .NET-to-.NET scenarios.

The central tension is coupling: MagicOnion buys enormous productivity for all-C# stacks by discarding gRPC's language neutrality. It also inherits every operational property of HTTP/2 gRPC (load-balancer requirements, TLS/ALPN handshakes, streaming lifetime management) while adding Unity-specific build constraints (AOT/IL2CPP code generation, code stripping) that plain gRPC users never encounter.

## Getting Started

```bash
# Server (ASP.NET Core Empty template)
dotnet add package MagicOnion.Server

# Client (.NET console, or Unity via NuGetForUnity / unitypackage)
dotnet add package MagicOnion.Client
```

```csharp
// Shared interface — referenced by BOTH server and client projects
public interface IMyFirstService : IService<IMyFirstService>
{
    UnaryResult<int> SumAsync(int x, int y);
}

// Server implementation
public class MyFirstService : ServiceBase<IMyFirstService>, IMyFirstService
{
    public async UnaryResult<int> SumAsync(int x, int y) => x + y;
}

// Server bootstrap (Program.cs)
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddMagicOnion();
var app = builder.Build();
app.MapMagicOnionService();
app.Run();

// Client
var channel = GrpcChannel.ForAddress("https://localhost:5001");
var client = MagicOnionClient.Create<IMyFirstService>(channel);
var result = await client.SumAsync(123, 456);
```

## Architecture / How It Works

Underneath, MagicOnion is a thin protocol layer over `grpc-dotnet` (`Grpc.Net.Client` on the client, ASP.NET Core's gRPC server on the host). Every Unary method compiles to a single gRPC method; the interface's namespace and method name become the gRPC service/method path, and MessagePack-serialized arguments become the request body. Because the schema is a C# type, there is no IDL to keep in sync — but the two sides must reference the same interface assembly (or a copy of it) and compatible MessagePack formatters.

Client proxies are produced two ways. On server and desktop .NET, MagicOnion can emit proxies at runtime via reflection/dynamic code generation. On platforms without runtime codegen — chiefly Unity IL2CPP and any AOT-compiled .NET — it relies on a **source generator** (`MagicOnion.Client.SourceGenerator`) that materializes the proxy and MessagePack formatters at compile time. Choosing the wrong path is the most common first-run failure on Unity.

**StreamingHub** is the more involved half. A hub is a long-lived object bound to one connection; clients call hub methods, and the server calls back through a strongly-typed `IReceiver` interface. Clients are organized into **groups**, and broadcasting to a group is the core realtime primitive. Within a single server process this is in-memory. Across multiple server instances it is not automatic: group broadcast to clients connected to *other* nodes requires an external backplane (Redis or NATS), historically via `MagicOnion.Server.Redis` and later Cysharp's Multicaster abstraction. Serialization is pluggable; MessagePack-CSharp is the default, with MemoryPack supported in recent versions.

## Production Notes

- **HTTP/2 all the way down.** Every gRPC caveat applies. Layer-4 (TCP) load balancers break gRPC connection multiplexing; you need an L7 proxy that understands HTTP/2 (Envoy, YARP, NGINX with gRPC, or a cloud gRPC LB). Naive round-robin at the connection level pins all streams from one client to one backend.
- **Unity IL2CPP and code stripping.** MessagePack formatters and MagicOnion proxies are generated types; aggressive managed-code stripping removes them, producing runtime `FormatterNotRegistered` / missing-type errors that do not appear in the Editor (Mono) but do in device builds (IL2CPP). Correct `link.xml` / source-generator setup is mandatory and is the dominant source of "works in Editor, crashes on device" reports.
- **Version lockstep.** Server and client must run compatible MagicOnion *and* MessagePack versions, and share the contract interface. Mismatched MessagePack major versions silently corrupt payloads. Treat the shared interface assembly as a versioned contract, not an afterthought.
- **StreamingHub lifetime.** Detecting a dead client on a long-lived duplex stream is not free; earlier versions were weak at noticing half-open connections. Heartbeat support (client and server) was added to make disconnect detection reliable — enable and tune it rather than assuming TCP will tell you.
- **Server platform floor.** The server requires .NET 8+. Clients span a much wider range (.NET Framework 4.6.1 through .NET 8+, .NET Standard 2.0/2.1, Unity 2022.3 LTS or newer). Upgrading the server across major MagicOnion versions has periodically raised that floor.
- **No built-in auth.** Authentication and authorization are done with gRPC interceptors / MagicOnion filters (attribute-based, similar to ASP.NET Core filters). There is no turnkey identity story; you wire JWT/session validation yourself.

## When to Use / When Not

**Use when:**
- Both server and client are C# — especially a .NET backend with a Unity game client.
- You want realtime group broadcast (matchmaking lobbies, game rooms, chat) without hand-rolling a WebSocket protocol.
- You value skipping `.proto` maintenance and want method calls that look like ordinary async C#.

**Avoid when:**
- Any client is non-.NET (JS/TS, Go, Python, mobile-native). Plain gRPC with `.proto`, or a REST/WebSocket API, will serve you better.
- You need a public, self-describing API contract for third parties — a C# interface is not a language-neutral schema.
- Your team is unwilling to own HTTP/2 load-balancing and Unity AOT build configuration; the operational surface is real.

## Alternatives

- grpc/grpc-dotnet — plain gRPC with `.proto` when you need genuine cross-language contracts instead of C#-only interfaces.
- protobuf-net/protobuf-net.Grpc — the closest analog: code-first gRPC from C# contracts, but without StreamingHub or Unity codegen focus.
- dotnet/aspnetcore (SignalR) — bidirectional realtime for browser/.NET clients when you don't need binary throughput or a Unity client.
- MirrorNetworking/Mirror — Unity-native high-level networking when you want transform/state synchronization rather than an RPC API surface.
- heroiclabs/nakama — a full game backend server (matchmaking, storage, presence) when you want batteries included rather than a framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2017 | First public releases; built on the native C-core gRPC library. |
| 4.x | ~2020–2021 | MessagePack-based contracts, Unity workflow maturity. |
| 5.0 | ~2022 | Migration from C-core gRPC to `grpc-dotnet` (`Grpc.Net.Client`)[^2]. |
| 6.x | ~2023 | StreamingHub and source-generator improvements. |
| 7.x | ~2024–2025 | Server floor raised to .NET 8; heartbeat, updated client source generator. |

## References

[^1]: MagicOnion README and documentation — "Unified Realtime/API framework for .NET platform and Unity." https://cysharp.github.io/MagicOnion/
[^2]: MagicOnion moved off the deprecated native C-core gRPC onto the managed `grpc-dotnet` stack, aligning with gRPC's own C# direction. https://cysharp.github.io/MagicOnion/
[^3]: Cysharp organization — UniTask, MessagePack-CSharp, MemoryPack, Multicaster. https://github.com/Cysharp

## Tags

csharp, dotnet, unity, grpc, rpc, realtime, streaming, messagepack, game-backend, networking, http2
