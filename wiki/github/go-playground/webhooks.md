# go-playground/webhooks

> A Go library that verifies and parses inbound webhook payloads from GitHub, GitLab, Bitbucket, Gogs, Docker Hub, and Azure DevOps into typed structs.

[GitHub repo](https://github.com/go-playground/webhooks) ·
[License: MIT](https://github.com/go-playground/webhooks/blob/master/LICENSE)

## Overview

`go-playground/webhooks` is a receive-side parsing library, not a webhook server or delivery framework. It gives you one thing per provider: a constructor that captures a shared secret, and a `Parse` method that takes an `*http.Request`, verifies the signature, matches the event type, and unmarshals the JSON body into a provider-specific payload struct. You supply the HTTP handler and the routing; the library supplies verification and typed decoding[^1].

Its design bet is exhaustive struct mapping: every field a provider sends is modeled as a Go struct field, so consumers get compile-time-checked access to the whole payload rather than fishing through `map[string]interface{}`[^1]. That is the library's value and its central tension — the structs are hand-maintained, so when a provider adds or renames fields the library lags until someone updates and releases the structs. Fields the library does not know about are silently dropped during unmarshal.

The project is part of the `go-playground` family (best known for `go-playground/validator`). It is MIT-licensed and has been stable for years; the current major version is v6, imported at `github.com/go-playground/webhooks/v6`[^2]. As of mid-2026 the repository shows roughly 1,000 stars and 240 forks, but its last push was in August 2024 — it is effectively in low-activity maintenance mode, which matters because a parser of external schemas needs ongoing updates to stay current with provider payloads.

## Getting Started

```shell
go get -u github.com/go-playground/webhooks/v6
```

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/go-playground/webhooks/v6/github"
)

func main() {
	hook, _ := github.New(github.Options.Secret("MyGitHubSuperSecretSecret...?"))

	http.HandleFunc("/webhooks", func(w http.ResponseWriter, r *http.Request) {
		payload, err := hook.Parse(r, github.ReleaseEvent, github.PullRequestEvent)
		if err != nil {
			if err == github.ErrEventNotFound {
				// event type wasn't in the list we asked Parse to handle
			}
			return
		}
		switch p := payload.(type) {
		case github.ReleasePayload:
			fmt.Printf("release: %+v\n", p)
		case github.PullRequestPayload:
			fmt.Printf("pull request: %+v\n", p)
		}
	})
	http.ListenAndServe(":3000", nil)
}
```

The `Parse` call both verifies the signature (when a secret was set) and filters: only the event types you pass are decoded, everything else returns `ErrEventNotFound`. You recover the concrete type with a type switch.

## Architecture / How It Works

Each provider lives in its own subpackage — `github`, `gitlab`, `bitbucket`, `gogs`, `dockerhub`, `azuredevops` — and each exposes the same shape: a `New(...Option)` constructor, a `Parse(r, events...)` method, an `Event` constant per event type, and a `...Payload` struct per event. There is no shared abstraction across providers; the uniformity is by convention, not by interface. If you handle two providers you instantiate two hooks and write two type switches.

`Parse` is synchronous and reads the entire request body into memory with `io.ReadAll` before unmarshalling — there is no streaming, so payload size maps directly to allocation. Verification is provider-specific: GitHub uses HMAC over the raw body compared against the `X-Hub-Signature-256` header; GitLab compares the `X-Gitlab-Token` header against the configured secret as a plain string; Bitbucket and others vary. This means the security guarantee differs by provider — the GitHub path is a cryptographic MAC, while the GitLab path is a shared-token equality check with weaker properties.

The hook value is stateless after construction (it holds only the secret and options), so a single hook is safe to share across concurrent requests. Only JSON payloads are supported; the library does not parse `application/x-www-form-urlencoded` deliveries, so a GitHub webhook must be configured with the JSON content type[^1]. Because decoding targets fixed structs, forward compatibility is bounded by the struct definitions in the installed version.

## Production Notes

- **Struct drift is the real operational risk.** Providers evolve their payloads continuously; this library's structs are updated manually and released on the maintainer's cadence. With the last commit in August 2024, expect newer fields (or entirely new event types) to be missing. If you depend on a field, assert its presence in tests against a captured real payload rather than trusting the struct.
- **Unknown fields are dropped, not errored.** A payload that decodes "successfully" can still be missing data your code assumes. There is no strict mode.
- **`Parse` consumes the body.** It reads `r.Body` to completion for verification and decoding; you cannot re-read it afterward, and any upstream middleware that already drained the body will break signature verification (the HMAC is computed over the exact bytes).
- **Verify the secret is actually set.** `New` without a `Secret` option produces a hook that skips verification — every unauthenticated POST parses. Treat a missing secret as a deployment error.
- **Version pinning matters more than usual.** Because correctness tracks external schemas, pin the module version and re-test after upgrades; a struct change between releases can shift field semantics.
- **Scope check before adopting.** This is a parser only: no delivery retries, no de-duplication of redelivered events, no idempotency keys, no async queueing. Those concerns are yours to build around it.

## When to Use / When Not

**Use when:**
- You receive webhooks from one or more of the supported providers and want typed structs plus signature verification without hand-rolling HMAC checks.
- You already have an HTTP server and just need the parse-and-verify layer.
- You value modeling the full payload over a minimal ad-hoc struct.

**Avoid when:**
- You only target GitHub and already use `google/go-github` — its built-in `ValidatePayload` / `ParseWebhook` cover the same ground with structs the same team keeps current.
- You need provider payloads that are guaranteed up to date; a lightly-maintained parser of moving schemas is a liability.
- You need form-encoded payloads, or webhook infrastructure (retries, dedup, fan-out) rather than parsing.

## Alternatives

- google/go-github — has `github.ValidatePayload` and `github.ParseWebhook`; use it when GitHub is your only provider and you want structs maintained alongside the API client.
- xanzy/go-gitlab — GitLab API client with its own webhook parsing; use it when GitLab is the focus.
- cbrgm/githubevents — an event-dispatch layer over go-github; use it when you want handler registration and routing, not just parsing.
- adnanh/webhook — a standalone configurable webhook server that runs commands; use it when you want a service, not a library, and no custom Go code.
- svix/svix-webhooks — signature verification helpers for the Svix delivery model; use it when you send or consume webhooks through Svix infrastructure.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-10-25 | Repository created; GitHub/Bitbucket/GitLab/Gogs receiver[^3]. |
| v6 (module) | — | Current major; import path `.../webhooks/v6`, adds Docker Hub and Azure DevOps support[^2]. |
| 6.3.0 | — | Version advertised by the README status badge[^1]. |
| last push | 2024-08-06 | Most recent commit to `master`; low-activity maintenance since[^3]. |

## References

[^1]: go-playground/webhooks README — features, JSON-only note, usage example, version badge. https://github.com/go-playground/webhooks
[^2]: Package documentation (v6 module path and provider subpackages). https://pkg.go.dev/github.com/go-playground/webhooks/v6
[^3]: GitHub repository metadata (created 2015-10-25, last push 2024-08-06, MIT, Go). https://github.com/go-playground/webhooks

## Tags

go, webhooks, github, gitlab, bitbucket, http, parser, signature-verification, library, devops
