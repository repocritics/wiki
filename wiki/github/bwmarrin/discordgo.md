# bwmarrin/discordgo

> Low-level Go bindings for the Discord API — a thin, near-complete mapping over the gateway, REST, and voice, with no framework on top.

[GitHub repo](https://github.com/bwmarrin/discordgo) ·
[Go reference](https://pkg.go.dev/github.com/bwmarrin/discordgo) ·
[License: BSD-3-Clause](https://github.com/bwmarrin/discordgo/blob/master/LICENSE)

## Overview

DiscordGo is a Go package providing bindings to the Discord chat API: the REST
endpoints, the real-time websocket gateway, and the UDP voice interface[^1]. It
has been maintained by Bruce Marriner (bwmarrin) since 2015 and is the
longest-lived Go Discord library. As of 2026 it has ~5.9k stars and ~920 forks,
with commits still landing but at a slow, maintenance-paced cadence rather than
rapid feature development.

The library's defining choice is stated explicitly in its own contributing
guide: it is intended to be a *low-level direct mapping* of the Discord API, and
enhancements outside that scope are discouraged[^1]. There is no command router,
argument parser, plugin system, sharding manager, or DI container. You get a
`Session`, typed structs for Discord objects, methods that call endpoints, and an
event handler registry. Everything above that — command frameworks, cooldowns,
pagination — is left to the application or third-party wrappers. This keeps
DiscordGo predictable and close to the wire, at the cost of more boilerplate than
batteries-included libraries in other languages. Because it tracks the API rather
than abstracting it, DiscordGo also inherits the API's churn directly: gateway
intents, the interactions (slash command) model, and message-content
restrictions each surfaced as additions application code had to adopt manually.

## Getting Started

```sh
go get github.com/bwmarrin/discordgo
```

```go
func main() {
	dg, err := discordgo.New("Bot " + os.Getenv("DISCORD_TOKEN"))
	if err != nil {
		panic(err)
	}

	// Gateway intents are mandatory; declare only what you use.
	dg.Identify.Intents = discordgo.IntentsGuildMessages | discordgo.IntentMessageContent

	dg.AddHandler(func(s *discordgo.Session, m *discordgo.MessageCreate) {
		if m.Author.Bot {
			return
		}
		if m.Content == "!ping" {
			s.ChannelMessageSend(m.ChannelID, "pong")
		}
	})

	if err := dg.Open(); err != nil { // opens the websocket
		panic(err)
	}
	defer dg.Close()

	select {} // block; real bots wait on an os.Signal channel
}
```

## Architecture / How It Works

`Session` is the central object and is safe for concurrent use. It holds the
websocket connection, the REST HTTP client, a rate-limiter, an optional in-memory
`State` cache, and the handler registry.

- **REST layer.** Each endpoint is a method (`ChannelMessageSend`,
  `GuildMemberAdd`, etc.) that marshals to JSON and calls Discord's HTTP API
  through a shared client. A built-in bucketed rate limiter respects Discord's
  per-route `X-RateLimit-*` headers and the global limit.
- **Gateway.** `Open()` establishes the websocket, sends the `Identify` payload
  (with your declared intents), and starts read/heartbeat goroutines. Incoming
  gateway events are unmarshaled into typed structs (`MessageCreate`,
  `GuildCreate`, `InteractionCreate`, …) and dispatched to handlers registered
  via `AddHandler`. Handler dispatch is reflection-based on the event's Go type.
- **State cache.** By default `Session.State` tracks guilds, channels, members,
  and roles from gateway events so lookups don't hit REST. It can be disabled
  (`StateEnabled = false`) to trade memory for freshness.
- **Voice.** `ChannelVoiceJoin` opens a separate voice websocket and a UDP
  socket, negotiates encryption, and exposes `OpusSend`/`OpusRecv` channels.
  DiscordGo does not encode Opus itself — you feed it pre-encoded Opus frames;
  helper projects `dgvoice` and `dca` wrap `ffmpeg` for that[^1].
- **Sharding.** Supported at the primitive level via `ShardID`/`ShardCount` on
  the `Session`; you construct one session per shard yourself. There is no
  automatic shard manager that spreads guilds for you.

Reconnection, heartbeat ACK tracking, and resume/replay are handled internally.
The dependency surface is small — historically `gorilla/websocket` plus
`golang.org/x/crypto` for voice encryption.

## Production Notes

- **Intents are non-optional and easy to get wrong.** If you don't request the
  right intent you silently receive no events; `IntentMessageContent` in
  particular is a *privileged* intent that must also be enabled in the Discord
  developer portal, or message text arrives empty. Symptoms look like a broken
  handler, not a permissions error.
- **`AddHandler` blocks the dispatch path.** Event handlers run on the gateway's
  read loop; a slow handler stalls processing of subsequent events. Offload
  anything blocking (DB calls, HTTP, long compute) to your own goroutine.
- **`master` vs. tags.** `go get` pulls the latest tagged release, but the
  README notes the library follows the still-evolving Discord API and warns of
  possible major changes[^1]. Some newer Discord features land on `master`
  before a tag; pinning to a commit is common for bleeding-edge endpoints.
- **Rate limits are per-route and global.** The built-in limiter handles the
  common case, but bulk operations (mass role edits, backfills) still need your
  own pacing; hitting the global limit pauses *all* requests on the session.
- **Voice is fragile.** Voice reconnection, region changes, and UDP hole-
  punching are the most-reported source of production issues; many teams pin
  voice-heavy bots to specific versions and keep `dca`/`dgvoice` at known-good
  commits.
- **No first-class command framework.** For slash commands you register
  `ApplicationCommand`s and switch on `InteractionCreate` yourself; for
  larger bots most teams adopt a community router or hand-roll one. Guild-scoped
  command registration is instant; global commands can take up to an hour to
  propagate (a Discord-side property, not a library bug).

## When to Use / When Not

**Use when:**
- You want a stable, low-level Go client that maps closely to the Discord API.
- You prefer explicit control over abstraction and don't mind writing your own
  command routing and lifecycle glue.
- You're building on Go's concurrency model and want a small dependency tree.

**Avoid when:**
- You want batteries-included ergonomics (command parsing, cooldowns, pagination)
  out of the box — you'll assemble or import those separately.
- You need a library that ships Discord's newest features the day they launch;
  DiscordGo's cadence is deliberate and sometimes trails the API.
- Your team is more comfortable in another ecosystem where the flagship library
  (discord.js, discord.py) carries a larger framework layer.

## Alternatives

- disgoorg/disgo — modern Go alternative, more opinionated, quicker to adopt new
  Discord features; use when you want a fuller framework in Go.
- diamondburned/arikawa — Go, cleanly separated gateway/state/API packages; use
  when you want a more composable architecture than DiscordGo's single Session.
- serenity-rs/serenity — Rust; use when you want memory-safety and a richer
  built-in command framework and are willing to leave Go.
- discordjs/discord.js — the reference Node.js library; use when your team lives
  in JavaScript/TypeScript and wants the largest ecosystem.
- Rapptz/discord.py — the Python standard; use when you want the most
  approachable batteries-included experience for smaller bots.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2015-11 | Repo created; early low-level bindings[^2]. |
| intents support | ~2020 | Gateway intents added as Discord made them mandatory. |
| interactions | ~2021 | Slash-command / `InteractionCreate` support (v0.23–0.24 era). |
| v0.27.x | 2023 | Continued API coverage and fixes. |
| v0.28.x | 2024 | Latest tagged line as of writing[^2]. |

## References

[^1]: DiscordGo README and contributing guidelines. https://github.com/bwmarrin/discordgo
[^2]: DiscordGo releases and tags. https://github.com/bwmarrin/discordgo/releases
[^3]: Discord API documentation — gateway, intents, and interactions. https://discord.com/developers/docs

## Tags

go, golang, discord, discord-api, bot, api-bindings, websocket, chat, library, voice
