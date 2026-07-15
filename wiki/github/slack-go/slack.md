# slack-go/slack

> The de facto Go client for the Slack platform — Web API, Socket Mode, Events API, and legacy RTM in one package.

[GitHub repo](https://github.com/slack-go/slack) ·
[Go reference](https://pkg.go.dev/github.com/slack-go/slack) ·
[License: BSD-2-Clause](https://github.com/slack-go/slack/blob/master/LICENSE)

## Overview

`slack-go/slack` is a Go library that wraps most of the `api.slack.com` REST surface, the Events API, Socket Mode, and the older Real-Time Messaging (RTM) websocket protocol. It began life as `nlopes/slack`, authored by Norberto Lopes, and was moved to the community-run `slack-go` organization around 2019 while keeping the same import path lineage[^1]. For most Go teams building Slack bots, slash-command handlers, or workspace automation, this is the library they reach for — there is no official first-party Slack SDK for Go, so this community package fills that role.

The defining tension is that Slack ships no supported Go SDK, so an unofficial library must chase a large, fast-moving, and inconsistently documented HTTP API. The maintainers have kept up remarkably well, but the project is explicit that it has never cut a `v1.0.0`: it lives permanently at `v0.x`, and the README warns that minor releases can carry backward-incompatible changes[^2]. That is the practical cost of tracking a moving target — you pin a version and read release notes on every bump.

The other structural fact worth internalizing is that Slack itself is mid-migration on its real-time story. RTM is deprecated on Slack's side in favor of Socket Mode and the Events API; the library still exposes RTM but nudges you toward Socket Mode, and its most modern ergonomic layer (`SocketmodeHandler`) is still labeled experimental[^2].

## Getting Started

```bash
go get -u github.com/slack-go/slack
```

```go
package main

import (
	"fmt"

	"github.com/slack-go/slack"
)

func main() {
	api := slack.New("xoxb-YOUR-BOT-TOKEN")
	// slack.OptionDebug(true) logs every request — useful when the API misbehaves.

	channelID, timestamp, err := api.PostMessage(
		"C0123456789",
		slack.MsgOptionText("deploy finished", false),
	)
	if err != nil {
		fmt.Printf("post failed: %s\n", err)
		return
	}
	fmt.Printf("sent to %s at %s\n", channelID, timestamp)
}
```

The constructor takes a token plus variadic `Option...` functions (`OptionDebug`, `OptionHTTPClient`, `OptionRetry`, `OptionAppLevelToken`, etc.), which is the pattern you configure everything through.

## Architecture / How It Works

The package is organized around one large `Client` struct returned by `slack.New`. Each Slack REST method is a method on that client (`PostMessage`, `GetUserInfo`, `GetConversations`, `UploadFileV2`, …), and the many optional parameters Slack accepts are expressed as functional options (e.g. `MsgOptionAttachments`, `GetUserGroupsOptionIncludeUsers`) rather than giant argument lists or config structs. Message composition uses the Block Kit types (`SectionBlock`, `ActionBlock`, `DividerBlock`, and the accessory/element types) that mirror Slack's JSON schema.

For receiving events there are three distinct paths, and choosing the right one matters:

- **Events API** (`slack/slackevents`) — you run an HTTP endpoint, Slack POSTs events to it, and you parse them with `slackevents.ParseEvent`. Requires a public URL. Best for stateless serverless/webhook deployments.
- **Socket Mode** (`slack/socketmode`) — an outbound websocket using an app-level token (`xapp-…`), so no public inbound URL is needed. This is Slack's recommended modern path and works behind firewalls. The lower-level `socketmode.Client` gives you a raw event channel; the higher-level `SocketmodeHandler` lets you register callbacks per event type, HTTP-router style, and is marked experimental[^2].
- **RTM** (`RTM` type via `api.NewRTM()`) — the legacy websocket protocol. Slack has deprecated it; new apps should not adopt it, but it remains in the library for existing integrations.

Interactivity (slash commands, shortcuts, modal submissions, block actions) is parsed via `slack.SlashCommandParse` and `InteractionCallback` decoding regardless of transport. The library is transport-agnostic about payloads: the same Block Kit and interaction types flow through Events API, Socket Mode, and RTM.

## Production Notes

- **Retries are off by default.** A raw `slack.New` does no automatic retry, so transient `429 Too Many Requests` responses surface as errors. Use `OptionRetry(n)` for 429-only retries or `OptionRetryConfig` for finer control; when combining with a custom client, pass retry options *after* `OptionHTTPClient`[^2]. Slack's rate limits are per-method tier, so a naive loop over `conversations.history` or `users.info` will get throttled quickly.
- **`v0.x` with breaking minors.** Because there is no stable major version, always pin an exact version in `go.mod` and read the release notes before upgrading. Method signatures and option types have changed across minor releases[^2].
- **File uploads changed API.** Slack deprecated the old `files.upload` endpoint; the library added `UploadFileV2` against the newer upload flow. Code still calling the older upload path should migrate, as the legacy endpoint is being retired on Slack's side.
- **Token types are easy to confuse.** Bot tokens (`xoxb-`), user tokens (`xoxp-`), and app-level tokens (`xapp-`) are not interchangeable. Socket Mode specifically needs an app-level token via `OptionAppLevelToken` in addition to the bot token, and mismatches produce opaque auth errors.
- **Pagination is manual.** Conversation, user, and history listings return cursors; you must loop on the `response_metadata.next_cursor` yourself. There is no automatic iterator, so forgetting the loop silently truncates results at the first page.
- **RTM is a dead end for new work.** Building on RTM today means adopting a protocol Slack is retiring; expect no new event types there and plan Socket Mode instead.

## When to Use / When Not

**Use when:**
- You are writing a Slack bot, slash-command handler, or workspace automation in Go and want broad coverage of the Web API in one dependency.
- You need Socket Mode or Events API handling with Block Kit message construction out of the box.
- You want a mature, widely-deployed library rather than hand-rolling HTTP calls against `api.slack.com`.

**Avoid when:**
- You need a formally stable, semver-`v1` dependency with a no-breaking-changes guarantee — this project deliberately stays `v0.x`.
- You only send the occasional notification: a single `chat.postMessage` HTTP call or an Incoming Webhook is lighter than pulling in the whole client.
- Your stack is not Go — Slack maintains official SDKs for JavaScript (Bolt), Python, and Java that are first-party supported.

## Alternatives

- slack/bolt-js — Slack's official first-party JS/TypeScript framework; use it when you are not committed to Go and want vendor-supported tooling.
- slackapi/bolt-python — official Python framework with the same first-party support; use when your automation lives in Python.
- slackapi/python-slack-sdk — lower-level official Python client; use when you want direct API access without Bolt's app framework.
- Incoming Webhooks (no library) — use when you only need one-way "post a message to a channel" notifications and don't want a dependency.
- Hand-rolled `net/http` calls — use when you touch one or two endpoints and want zero third-party code.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial `nlopes/slack` | 2015-01 | Repository created by Norberto Lopes; RTM + Web API[^1]. |
| Moved to `slack-go` org | ~2019 | Transferred to community `slack-go` organization[^1]. |
| Socket Mode support | 2021 | `socketmode` package added after Slack launched Socket Mode. |
| `SocketmodeHandler` | later | Higher-level routed event handler, marked experimental[^2]. |
| `UploadFileV2` | 2024 | New upload flow after Slack deprecated `files.upload`. |
| Ongoing `v0.x` | 2026 | Still no `v1.0.0`; minors may break compatibility[^2]. |

## References

[^1]: Repository history — `slack-go/slack` (formerly `nlopes/slack`), created 2015-01-24. https://github.com/slack-go/slack
[^2]: Project README — status, retries, Socket Mode handler, and versioning notes. https://github.com/slack-go/slack/blob/master/README.md
[^3]: Slack API rate limits and method tiers. https://api.slack.com/docs/rate-limits
[^4]: Slack Socket Mode documentation. https://api.slack.com/apis/socket-mode

## Tags

go, golang, slack, slack-api, chatbot, socket-mode, events-api, api-client, bot, sdk
