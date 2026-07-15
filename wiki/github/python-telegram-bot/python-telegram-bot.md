# python-telegram-bot/python-telegram-bot

> A pure-Python async wrapper over the Telegram Bot API, plus a higher-level handler/application framework in `telegram.ext`.

[GitHub repo](https://github.com/python-telegram-bot/python-telegram-bot) ·
[Official website](https://python-telegram-bot.org) ·
[License: LGPL-3.0](https://www.gnu.org/licenses/lgpl-3.0.html)

## Overview

python-telegram-bot (PTB) is the oldest and most-used Python library for the Telegram Bot API, first published in 2015[^1]. It has two layers that are often confused. The bottom layer (`telegram`) is a faithful, fully type-annotated object model of the Bot API — one class per API type, one method per API endpoint — over an HTTP client. The top layer (`telegram.ext`) is an opinionated application framework: an update queue, a handler-dispatch system, conversation state machines, a job scheduler, and persistence. You can use the bottom layer alone as a thin API client, but most users adopt the framework.

The library's defining event is the **v20 rewrite (January 2023)**, which converted the entire codebase from threaded/synchronous to `asyncio` and replaced the networking stack[^2]. This was a hard break: essentially no v13 code runs on v20 unchanged, and the pre-v20 `Updater`/`Dispatcher` entry point was replaced by `Application`. Because PTB has a decade of tutorials, Stack Overflow answers, and blog posts indexed online, the single largest source of confusion for newcomers is copying synchronous v13-era code into a v20+ project. Pin your version and check the docs match it before trusting any example.

PTB tracks the Bot API closely — the README currently advertises Bot API 10.0 support[^3] — and the maintainers backfill new Telegram features quickly after each API release. It is a hobby-maintained project (the team explicitly declines donations[^1]) but has stayed consistently active for its entire history.

## Getting Started

```bash
pip install python-telegram-bot --upgrade
# framework extras (rate limiter, webhooks, callback-data, job-queue):
pip install "python-telegram-bot[ext]"
```

```python
# echo_bot.py — telegram.ext framework, async
from telegram import Update
from telegram.ext import Application, MessageHandler, filters, ContextTypes

async def echo(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(update.message.text)

def main() -> None:
    app = Application.builder().token("YOUR_TOKEN").build()
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, echo))
    app.run_polling()  # blocks; long-polls getUpdates

if __name__ == "__main__":
    main()
```

Handlers are `async` coroutines. `run_polling()` and `run_webhook()` own the event loop and lifecycle; do not create your own loop around them.

## Architecture / How It Works

The two layers map to two import roots:

- **`telegram`** — the Bot API surface. `Bot` holds one method per endpoint (`send_message`, `get_updates`, …), returning modeled objects (`Message`, `Update`, `Chat`, `User`). Everything is annotated and immutable-ish; objects are built from and serialize back to Telegram's JSON. `HTTPXRequest` (over `httpx`, the only required dependency) is the default transport and is swappable.
- **`telegram.ext`** — the framework. `Application` is the central object (it replaced v13's `Dispatcher`). `Updater` now has a narrowed job: fetch updates via long-polling or a webhook server and push them onto `Application.update_queue`. The `Application` pulls from that queue and routes each update through registered **handlers** (`CommandHandler`, `MessageHandler`, `CallbackQueryHandler`, `ConversationHandler`, …) matched by **filters**.

Key subsystems, all in `telegram.ext`:

- **ConversationHandler** — a per-user/per-chat state machine for multi-step dialogs. Powerful but the most error-prone piece: the `per_chat`/`per_user`/`per_message` keying rules and interaction with persistence trip up most users.
- **JobQueue** — scheduled/recurring callbacks, backed by APScheduler (optional dependency since v20).
- **Persistence** — `BasePersistence` with a `PicklePersistence` default; stores `user_data`/`chat_data`/`bot_data` and conversation states across restarts.
- **Rate limiting** — `AIORateLimiter` (optional, via `aiolimiter`) queues outgoing calls under Telegram's limits.
- **CallbackDataCache** — "arbitrary callback_data": stores real Python objects server-side and sends only a UUID in inline-button payloads, working around Telegram's 64-byte callback-data cap.

The whole framework runs on a **single asyncio event loop and is not thread-safe by design**[^4]. Concurrency comes from `await`, not threads: handlers run sequentially per update by default; you opt into overlap with `concurrent_updates=True` or per-handler `block=False`.

## Production Notes

- **One consumer per bot token.** Telegram's `getUpdates` allows only one active long-poller per token, so a polling bot does not horizontally scale by running more instances — the second instance gets conflict errors. Webhooks fan out better but you still coordinate a single logical bot. Sharding means multiple bot tokens, not multiple replicas.
- **Blocking calls freeze everything.** Because it is single-loop, any synchronous blocking call inside a handler (a `requests` call, heavy CPU, `time.sleep`) stalls the whole bot. Offload to `asyncio.to_thread`/executors or use async clients.
- **Telegram's own rate limits are the real ceiling:** roughly 30 messages/second globally and about 1 message/second to a given chat (bursts tolerated, sustained bursts hit `RetryAfter`). Handle `telegram.error.RetryAfter` and consider `AIORateLimiter`; the library does not silently throttle for you unless you enable it.
- **Persistence is not multi-process safe.** `PicklePersistence` is a single-file store; concurrent writers corrupt it. For multi-worker or restart-durable deployments, implement `BasePersistence` over Redis/SQL yourself (community backends exist but are not first-party).
- **Version pinning is mandatory.** The v20 rewrite means examples, extensions, and answers are split across two incompatible eras. Optional features live behind extras (`[webhooks]` pulls `tornado`, `[job-queue]` pulls APScheduler, `[callback-data]` pulls cachetools); a missing extra surfaces as an import-time error, not a clear message.
- **License is LGPL-3.0, not GPL-3.0.** GitHub's license detector reports GPL-3.0[^5], but the README, PyPI metadata, and `LICENSE.lesser` state LGPLv3: you may ship closed applications that *use* PTB, but modifications to PTB itself must stay LGPL. Verify against the actual license files if this matters legally.
- **Free-threaded Python 3.14** is tested but explicitly not guaranteed thread-safe (tracking issue #4873)[^4].

## When to Use / When Not

**Use when:**
- You are building a Telegram *bot* (not automating a user account) in Python and want the most mature, best-documented option.
- You want a batteries-included framework: handlers, conversation state, jobs, persistence, and rate limiting out of one package.
- You value a fully type-annotated API and close tracking of new Bot API versions.

**Avoid when:**
- You need MTProto capabilities — user-account automation, reading full chat history, large-file transfer beyond bot limits. The Bot API (and thus PTB) cannot do these; use an MTProto client.
- You want a router/FSM-first design or ASGI-native integration — aiogram fits that style more naturally.
- You are locked to legacy synchronous code and cannot adopt `asyncio`; the last sync series (v13) is unmaintained.

## Alternatives

- aiogram/aiogram — async Bot API framework with built-in FSM and router-based dispatch; use instead when you prefer its more modern, declarative structure.
- pyrogram/pyrogram — MTProto client; use instead when you need user-account automation or capabilities the Bot API does not expose.
- LonamiWebs/Telethon — mature MTProto library for full Telegram client access; use instead when you need bot *and* user sessions with deep protocol control.
- eternnoir/pyTelegramBotAPI — simpler, lighter Bot API wrapper (sync and async); use instead when you want minimal abstraction over the raw API.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-07 | First public release; threaded, urllib3-based Bot API wrapper[^1]. |
| 12.0 | 2019-05 | Large `telegram.ext` overhaul; still synchronous. |
| 13.0 | 2020-10 | Final synchronous/threaded major series. |
| 20.0 | 2023-01 | Full `asyncio` rewrite; `httpx` transport; `Application` replaces `Dispatcher`; hard break from v13[^2]. |
| 21.x | 2024 | Ongoing async series; sigstore-signed releases from v21.4[^1]. |
| current | 2026 | Native support for Bot API 10.0; Python 3.10+[^3]. |

## References

[^1]: python-telegram-bot README and project site. https://github.com/python-telegram-bot/python-telegram-bot and https://python-telegram-bot.org
[^2]: PTB v20 transition guide (async rewrite, `Application` vs `Updater`/`Dispatcher`). https://github.com/python-telegram-bot/python-telegram-bot/wiki/Transition-guide-to-Version-20.0
[^3]: README "Telegram API support" — Bot API 10.0, Python 3.10+ (fetched 2026-07). https://github.com/python-telegram-bot/python-telegram-bot
[^4]: README "Concurrency" / "Free threading" sections; free-threading tracking issue #4873. https://github.com/python-telegram-bot/python-telegram-bot/issues/4873
[^5]: GitHub repository API reports `GPL-3.0`; README License section and PyPI metadata state LGPL-3.0. https://www.gnu.org/licenses/lgpl-3.0.html

## Tags

python, telegram, telegram-bot, chatbot, bot-framework, asyncio, bot-api, httpx, api-wrapper, async
