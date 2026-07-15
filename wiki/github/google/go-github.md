# google/go-github

> Go client library for the GitHub REST API v3 — service-grouped, pointer-heavy, and versioned in lockstep with GitHub's own API surface.

[GitHub repo](https://github.com/google/go-github) ·
[Package docs](https://pkg.go.dev/github.com/google/go-github/v89/github) ·
[License: BSD-3-Clause](https://github.com/google/go-github/blob/master/LICENSE)

## Overview

go-github is a Go client for the GitHub REST API v3. It was started by Will Norris in 2013 and lives under the `google` GitHub organization, though it is explicitly "not an official Google product"[^1]. It is the de facto standard Go binding for GitHub: most Go tooling that talks to GitHub — CI bots, release automation, GitHub Apps, security scanners — sits on top of it or a fork of it.

The library is a thin, near-mechanical mapping of the REST API into Go. Endpoints are grouped into services (`client.Repositories`, `client.Issues`, `client.PullRequests`, `client.Actions`, `client.Organizations`, and dozens more) that mirror the structure of GitHub's REST documentation. There is almost no business logic: a call sends an HTTP request, decodes JSON into a struct, and returns `(value, *github.Response, error)`. The `Response` carries pagination cursors and rate-limit state alongside the standard `http.Response`.

Two design decisions define the day-to-day experience. First, **every non-repeated struct field is a pointer** (`*string`, `*bool`, `*int`), so the library can distinguish "field absent" from "field set to zero"[^2] — precise but verbose, and a frequent source of nil-dereference panics. Second, the **major version is baked into the import path** (`.../go-github/v89/github`), and go-github ships major releases roughly monthly because any incompatible change to GitHub's API surface bumps the major[^3]. Staying current means editing import paths often.

This library covers only the REST API v3. For the GraphQL API v4, the maintainers point you to a different project (shurcooL/githubv4)[^1].

## Getting Started

```bash
go get github.com/google/go-github/v89
```

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/google/go-github/v89/github"
)

func main() {
	// Unauthenticated client, or use github.WithAuthToken("...") for a PAT.
	client := github.NewClient(nil).WithAuthToken("... your token ...")

	opt := &github.RepositoryListByOrgOptions{
		Type:        "public",
		ListOptions: github.ListOptions{PerPage: 50},
	}

	var all []*github.Repository
	for {
		repos, resp, err := client.Repositories.ListByOrg(context.Background(), "golang", opt)
		if err != nil {
			log.Fatal(err)
		}
		all = append(all, repos...)
		if resp.NextPage == 0 {
			break
		}
		opt.Page = resp.NextPage
	}
	fmt.Printf("%d repos\n", len(all))
}
```

Newer releases also expose `github.NewClient(github.WithAuthToken(...))` functional options and, on Go 1.23+, `iter`-based `*Iter` methods that hide the pagination loop.

## Architecture / How It Works

The client is a single `github.Client` struct holding an `*http.Client`, a base URL, and one field per service. Services are just typed views over the same shared client; they hold no state of their own. All authentication is delegated to the HTTP layer — go-github never handles tokens directly. You either pass a token via `WithAuthToken` (which installs a transport that adds the `Authorization` header) or supply your own `http.RoundTripper` via `WithTransport`. This is why GitHub App auth is done with a *separate* library such as bradleyfalzon/ghinstallation whose `Transport` you hand to the client — the App/installation JWT dance is out of scope for go-github itself.

Much of the codebase is generated. Per-field accessor methods (`GetName()`, `GetPrivate()` — nil-safe getters that return the zero value when the pointer is nil) live in a generated `github-accessors.go`, and query-string encoding of `...Options` structs is done via `google/go-querystring` reflection over `url` struct tags. This keeps the hand-written surface small and uniform, which is why "add a new endpoint" is a mechanical, well-documented contribution.

Pagination comes in two flavors: page-number (`ListOptions{Page, PerPage}`, with `resp.NextPage`) and opaque cursor (`ListCursorOptions{After}`). Errors are typed: `*github.ErrorResponse` for 4xx/5xx with GitHub's error body, `*github.RateLimitError` for primary limits, `*github.AbuseRateLimitError` for secondary limits, and `*github.AcceptedError` for 202 responses where GitHub is computing the answer asynchronously (contributor stats, for example). You discriminate them with `errors.As`.

Webhooks are a self-contained corner: `ValidatePayload` verifies the HMAC signature, and `ParseWebHook` unmarshals the JSON into one of the many `*Event` structs, which you then type-switch over.

## Production Notes

**Major-version churn is the dominant operational cost.** Because the major version is in the import path and bumps ~monthly, a codebase that pins `v60` will silently fall behind, and moving to `v89` is a find-and-replace across every import plus a build to catch renamed or removed symbols. Teams often wrap the client behind an internal package to localize the upgrade blast radius. Read the generated release notes (the repo ships a `gen-release-notes` tool) before each bump.

**Pointer fields cause panics, not compile errors.** Reading `*repo.Description` when the field is nil panics at runtime. Prefer the generated nil-safe getters (`repo.GetDescription()`); use raw pointer access only when you specifically need to detect absence. When constructing request bodies, `github.Ptr(...)` (the unified helper; older code uses `github.String`/`Bool`/`Int`) builds the pointers.

**Rate limiting is surfaced, not managed.** The client tracks the last-seen rate limit in `Response.Rate` and can optionally sleep until reset (`SleepUntilPrimaryRateLimitResetWhenRateLimited` context value), but for serious throughput you want a dedicated middleware transport such as gofri/go-github-ratelimit that handles both primary and secondary limits with backoff. Secondary (abuse) limits are triggered by concurrency and burst, not just total count, and are easy to hit with parallel workers.

**Conditional requests are your responsibility.** go-github does not cache. To exploit `ETag`/`If-None-Match` (304s that don't count against your rate limit), wrap the client's HTTP transport in an RFC 9111 cache (e.g. bartventer/httpcache). This is often the single biggest lever for high-volume read workloads.

**Preview/experimental API coverage is intentionally unstable.** Minor releases can change preview-endpoint behavior without warning, because GitHub itself does — the versioning policy treats preview functionality as non-stable[^3].

**Testing.** The verbose pointer structs make hand-written fixtures painful; migueleliasweb/go-github-mock provides a mock HTTP layer that most projects reach for instead of stubbing the client.

## When to Use / When Not

**Use when:**
- You are writing Go and need the GitHub REST API v3 with full, current endpoint coverage.
- You want a low-magic client where behavior maps directly to the documented HTTP API.
- You are building GitHub Apps, webhook receivers, or automation and want typed events and errors.

**Avoid when:**
- You need the GraphQL API v4 — use shurcooL/githubv4 instead.
- You cannot tolerate frequent import-path-breaking upgrades and want a library that changes rarely.
- You only need a couple of endpoints in a short-lived script — a raw `http` call may be lighter than pulling in the pointer-struct model.
- You are building a `gh` CLI extension, where cli/go-gh reuses the user's existing `gh` auth and config.

## Alternatives

- shurcooL/githubv4 — use instead when you need GitHub's GraphQL v4 API rather than REST v3 (recommended by go-github's own maintainers).
- cli/go-gh — use instead when building `gh` CLI extensions; it reads auth/host from existing `gh` config and is lighter for CLI contexts.
- migueleliasweb/go-github-mock — complementary, not a replacement; use for mocking go-github responses in tests.
- octokit/octokit.js — use instead when your stack is JavaScript/TypeScript rather than Go.
- PyGithub/PyGithub — use instead when you are in Python and want a similarly REST-oriented client.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2013-05 | Created by Will Norris; single import path, GOPATH era.[^1] |
| Modules | ~2018 | Adopted Go modules and semantic import versioning (`/vN` in path). |
| ongoing | 2018–present | Roughly monthly major releases, each tracking GitHub API changes.[^3] |
| Functional-options client | recent | `NewClient(WithAuthToken(...))` / `WithTransport(...)` option style. |
| Iterators | recent | Go 1.23 `iter`-based `*Iter` pagination helpers added. |
| v89 | 2026 | Current release; import path `.../go-github/v89/github`. |

## References

[^1]: go-github README — project scope, GraphQL recommendation, and "not an official Google product" notice. https://github.com/google/go-github/blob/master/README.md
[^2]: go-github README, "Creating and Updating Resources" — rationale for pointer fields and helper constructors. https://github.com/google/go-github/blob/master/README.md#creating-and-updating-resources
[^3]: go-github README, "Versioning" — major-version-in-import-path policy and preview-feature instability. https://github.com/google/go-github/blob/master/README.md#versioning

## Tags

go, golang, github-api, rest-api, client-library, sdk, api-wrapper, webhooks, github-apps, pagination
