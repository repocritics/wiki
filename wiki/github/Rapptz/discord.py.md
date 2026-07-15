# Rapptz/discord.py

> An async-first Python wrapper for the Discord bot API — the reference implementation most other Python Discord libraries were forked from.

[GitHub repo](https://github.com/Rapptz/discord.py) ·
[Documentation](https://discordpy.readthedocs.io/en/latest/) ·
[License: MIT](https://github.com/Rapptz/discord.py/blob/master/LICENSE)

## Overview

discord.py is a Python library for building Discord bots, written by Danny (Rapptz) and first released in 2015[^1]. It wraps Discord's two transport surfaces — a persistent WebSocket "gateway" for receiving events and a REST API for actions — behind an `async`/`await` event model. It is the oldest and most widely used Python Discord library, and the codebase from which the current generation of alternatives (Pycord, nextcord, disnake) was forked.

The defining episode in its history is the **2021 discontinuation and 2022 revival**. In August 2021 the author announced he would stop maintaining the project, citing Discord's move to make message content a privileged intent and to push interaction-based commands (slash commands) as the sanctioned model[^2]. Development resumed in early 2022 and shipped as v2.0, which added first-class support for application commands, message components (buttons, select menus, modals), and dropped Python 3.7[^3]. Several forks created during the hiatus continued independently and still exist, so newcomers routinely encounter tutorials written against incompatible APIs.

The central tension is that discord.py tracks a fast-moving, closed platform API it does not control. Discord ships gateway/intent/interaction changes on its own schedule; the library absorbs them across major versions, and bots written for the pre-2.0 "message command" world require real migration effort.

## Getting Started

```sh
# base install (Python 3.8+)
python3 -m pip install -U discord.py

# with voice support (pulls PyNaCl; needs libopus + libffi at runtime)
python3 -m pip install -U "discord.py[voice]"
```

```python
import discord
from discord.ext import commands

intents = discord.Intents.default()
intents.message_content = True   # privileged — must also be enabled in the Developer Portal

bot = commands.Bot(command_prefix=">", intents=intents)

@bot.event
async def on_ready():
    print(f"Logged in as {bot.user}")

@bot.command()
async def ping(ctx):
    await ctx.send("pong")

bot.run("TOKEN")
```

Slash commands use the app-command tree instead of the prefix router:

```python
@bot.tree.command(description="Say hello")
async def hello(interaction: discord.Interaction):
    await interaction.response.send_message("hi")

# app commands must be synced to Discord once, e.g. in on_ready:
# await bot.tree.sync()
```

## Architecture / How It Works

The library is built on `aiohttp` and a single asyncio event loop. Two subsystems do the real work:

- **Gateway client** — a WebSocket connection that receives events (message create, member join, reaction, etc.). The library maintains the heartbeat, handles reconnect/resume, and dispatches decoded events to your coroutine handlers. Large bots (Discord requires sharding past ~2,500 guilds) split the gateway across shards; `AutoShardedClient` manages this.
- **HTTP client** — a REST layer with built-in rate-limit handling. Discord returns per-route buckets and a global limit; discord.py queues and delays requests to respect both, so most bots never think about 429s until they hit unusual bulk-operation patterns.

State is cached in memory. The library hydrates a local object graph (guilds, channels, members, roles) from gateway events, so attribute access like `guild.members` is served from cache rather than an API call. This cache is populated according to **intents** — a bitmask, introduced by Discord, that gates which events a bot receives. `members`, `presences`, and `message_content` are *privileged* intents that must be toggled in the Developer Portal (and, past 100 servers, verified). Forgetting to enable an intent is the single most common "why is this list empty / why is `message.content` blank" bug.

Two bundled extensions sit on top of the core client:

- **`discord.ext.commands`** — the prefix-command framework (`Bot`, `Cog`, converters, checks, the `Context` object). Predates slash commands and remains widely used.
- **`discord.ext.tasks`** — a scheduler for recurring background loops with backoff.

Application commands (`bot.tree`) are a parallel, newer surface layered beside the prefix framework rather than replacing it; many production bots run both.

## Production Notes

- **Intents are a deploy-time footgun, not a code bug.** The most frequent failure is a bot that runs fine locally but sees empty member lists or blank message content in production because the privileged intent was never enabled in the portal. Enabling it in code (`intents.message_content = True`) is necessary but not sufficient.
- **Command syncing is rate-limited and easy to misuse.** `tree.sync()` registers slash commands globally and global sync can take up to an hour to propagate; guild-scoped sync is instant. Calling `sync()` on every startup risks hitting Discord's command-registration limits — sync deliberately, not on each boot.
- **Blocking the event loop stalls everything.** A synchronous CPU-bound call or a blocking library (`requests`, `time.sleep`) inside a handler freezes the heartbeat and can drop the gateway connection. Use `aiohttp`/async equivalents or offload to `run_in_executor`.
- **Cache lives in RAM and grows with scale.** Member/message caches are memory-resident; large bots tune `max_messages`, `chunk_guilds_at_startup`, and `member_cache_flags` to bound footprint. There is no built-in persistence layer — you bring your own database.
- **Voice is fragile.** Voice requires PyNaCl and a system libopus, is sensitive to network conditions, and reconnect handling around voice has historically been a rough edge. Many bots that only need audio playback lean on external tools rather than in-process voice.
- **Migration cost is real.** Pre-2.0 code (message-content-based commands, older event signatures) does not run unmodified on 2.x. Tutorials and Stack Overflow answers predating 2022 are frequently wrong for current versions — check that examples target 2.x before copying.

## When to Use / When Not

**Use when:**
- You are building a Discord bot in Python and want the most-documented, most-supported library.
- You want both classic prefix commands and modern slash commands available in one framework.
- You value a mature rate-limit and reconnect implementation you don't have to write.

**Avoid when:**
- You need a non-Python stack — discord.js (JavaScript) is larger and closer to Discord's own examples.
- You want a lower-level, unopinionated core without the bundled command framework — hikari targets that niche.
- You are locked to a fork's exclusive feature or an existing fork-based codebase; migrating between discord.py and its forks is non-trivial despite shared ancestry.

## Alternatives

- Pycord/pycord — discord.py fork from the 2021 hiatus; similar API, shipped some features earlier. Use when a tutorial or codebase already targets it.
- DisnakeDev/disnake — another hiatus-era fork with its own interaction API additions. Use when you need its specific extensions.
- nextcord/nextcord — hiatus-era fork focused on staying close to the original API. Use for a familiar surface with independent maintenance.
- hikari-py/hikari — ground-up, more explicit async Discord library (not a fork). Use when you want a lower-level core and to pick your own command framework.
- discordjs/discord.js — the JavaScript equivalent. Use when your stack is Node rather than Python.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015 | Initial releases; early async API[^1]. |
| 1.0 | 2019 | Model rewrite; stricter typed object graph. |
| — | 2021-08 | Author announces discontinuation over privileged intents / interactions[^2]. |
| 2.0 | 2022-08 | Development resumed; app commands, UI components, Python 3.8+[^3]. |
| 2.1–2.4 | 2023–2024 | Incremental interaction, component, and API coverage. |
| 2.5 | 2025 | Continued API tracking and fixes. |

## References

[^1]: Rapptz/discord.py repository — created 2015-08-21. https://github.com/Rapptz/discord.py
[^2]: Danny (Rapptz), "discord.py — Yielding to the Future" (discontinuation announcement), gist, 2021-08. https://gist.github.com/Rapptz/4a2f62751b9600a31a0d3c78100287f1
[^3]: discord.py changelog / migration to v2.0. https://discordpy.readthedocs.io/en/stable/migrating.html
[^4]: Discord API — Gateway intents and privileged intents. https://discord.com/developers/docs/topics/gateway#gateway-intents

## Tags

python, discord, discord-bot, api-wrapper, asyncio, aiohttp, chatbot, bot-framework, websocket, slash-commands
