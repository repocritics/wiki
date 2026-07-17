# markbates/goth

> Multi-provider OAuth/OAuth2 authentication for Go, built around two small interfaces instead of a framework.

[GitHub repo](https://github.com/markbates/goth) ·
[Maintainer note](https://blog.gobuffalo.io/goth-needs-a-new-maintainer-626cd47ca37b) ·
[License: MIT](https://github.com/markbates/goth/blob/master/LICENSE.txt)

## Overview

Goth is a Go library that standardizes "log in with X" flows across 50+ identity
providers (Google, GitHub, Facebook, Apple, Discord, Okta, generic OpenID
Connect with auto-discovery, and many more) behind a single pair of interfaces.
It was inspired by Ruby's `omniauth`[^1] and follows the same philosophy: the
core defines the contract, and each provider is a small, self-contained adapter.
The repo dates to 2014 and remains one of the most-referenced auth packages in
the Go ecosystem[^2].

Its defining design choice is that Goth is *not* a framework and does not own
your HTTP handlers. The root package (`goth`) is transport-agnostic: it knows
only how to begin an auth flow, complete it, and return a normalized `User`
struct. A separate sub-package, `gothic`, provides the `net/http` glue — session
storage, state handling, and the two handler functions most apps call. This
split is the source of both its flexibility and most of its footguns.

The honest tension: Goth optimizes for breadth of providers and a minimal core,
not for turnkey security defaults. It gives you correct OAuth plumbing and a
normalized user, but leaves session security, token refresh, and CSRF posture
largely to you. It is a login-flow library, not an identity or session-management
platform.

## Getting Started

```bash
go get github.com/markbates/goth
```

```go
import (
	"fmt"
	"net/http"

	"github.com/markbates/goth"
	"github.com/markbates/goth/gothic"
	"github.com/markbates/goth/providers/github"
)

func main() {
	goth.UseProviders(github.New(
		"CLIENT_ID", "CLIENT_SECRET",
		"http://localhost:3000/auth/github/callback"))

	http.HandleFunc("/auth/github", gothic.BeginAuthHandler)
	http.HandleFunc("/auth/github/callback", func(w http.ResponseWriter, r *http.Request) {
		user, err := gothic.CompleteUserAuth(w, r) // reads provider from request
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "Hello %s (%s)", user.Name, user.Email)
	})
	http.ListenAndServe(":3000", nil)
}
```

## Architecture / How It Works

The entire model rests on two interfaces[^3]:

- **`Provider`** — `BeginAuth`, `UnmarshalSession`, `FetchUser`, `Name`, plus
  refresh-token methods. Each provider under `providers/<name>/` implements this.
- **`Session`** — `GetAuthURL`, `Marshal`, `Authorize`. A provider's session
  carries the flow's state (tokens, verifier) and is serialized between the
  redirect and the callback.

`goth.UseProviders(...)` registers providers into a global map. The OAuth2
providers wrap `golang.org/x/oauth2` for the actual token exchange; OAuth1 and
bespoke providers (Twitter, Steam) implement it directly. Because a `Session` is
marshaled to a string and stashed between two requests, Goth needs somewhere to
put it — that is `gothic`'s job.

`gothic` stores the in-flight session using `gorilla/sessions`, defaulting to a
`CookieStore`[^4]. `BeginAuthHandler` marshals the provider session into the
cookie and redirects; `CompleteUserAuth` reads it back, completes the exchange,
and returns the normalized `goth.User`. The provider is selected from a request
parameter named `provider` (via `gothic.GetProviderName`, overridable), which is
why examples route as `/auth/{provider}` and `/auth/{provider}/callback`.

The global provider registry and the package-level `gothic.Store` variable make
configuration process-global mutable state, not dependency-injected — idiomatic
to the library's era, but it complicates testing and multi-tenant setups.

## Production Notes

- **Insecure cookie default.** The out-of-the-box `gothic.Store` sets
  `Secure: false` and a 30-day `MaxAge`[^4]. You must override the store at
  startup to set `Secure: true` (HTTPS), tune `MaxAge`, and — critically —
  supply a strong key from your own secret. Shipping the defaults over plain
  HTTP leaks the session cookie.
- **`SESSION_SECRET` is mandatory in practice.** `gothic`'s default store is
  seeded from the `SESSION_SECRET` env var. An empty or weak secret means
  forgeable cookies — easy to miss, since the app still boots without it.
- **Cookie size limits.** The marshaled provider session lives in the cookie by
  default, so providers returning large tokens/claims (some OIDC setups) can
  exceed the ~4KB limit. Swap `gothic.Store` for a server-side backend then.
- **Token refresh is provider-dependent.** Goth exposes `RefreshToken` where the
  provider supports it, but not all do, and it does not manage token lifecycle or
  persistence — long-lived access requires your own storage and refresh logic.
- **State/CSRF protection depends on the cookie round-trip.** Misconfigured or
  insecure stores weaken it; validate store settings as part of security review.
- **Maintenance posture.** The project's own homepage links to a "goth needs a
  new maintainer" post[^5]; upkeep has long been community/gobuffalo-driven rather
  than actively feature-developed. Expect a stable, slow-moving core: new
  providers arrive via PRs, but issue response is slow. Pin a version and read
  provider code before depending on it. Its `gorilla/sessions` dependency also
  went through archival before the Gorilla toolkit was revived — verify releases.

## When to Use / When Not

**Use when:**
- You need "log in with X" across many providers and want one normalized `User`.
- You want OAuth plumbing without adopting a full auth framework or SaaS.
- You need a provider Goth already ships, or you can write a ~200-line adapter.
- You control your own session and want the login flow to stay out of the way.

**Avoid when:**
- You want secure-by-default sessions and managed tokens out of the box — you
  will have to configure the store correctly yourself.
- You need full identity management (user DB, RBAC, MFA, refresh orchestration).
- You want an actively feature-developed dependency with fast issue turnaround.
- You are on a framework with its own first-class auth story you'd be fighting.

## Alternatives

- coreos/go-oidc — use when you only need standards-based OpenID Connect against
  one IdP and want a smaller, focused dependency.
- golang.org/x/oauth2 — use when you want to hand-roll a single provider's OAuth2
  flow and skip the abstraction entirely.
- ory/kratos — use when you need a full self-hosted identity system (registration,
  recovery, MFA), not just social login.
- authelia/authelia — use when you want an authentication gateway/portal in front
  of apps rather than an in-process library.
- zitadel/zitadel — use when you'd rather delegate identity to a self-hostable
  OIDC provider and integrate via standard flows.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2014-10-14 | Repo created; core `Provider`/`Session` model inspired by `omniauth`[^1][^2]. |
| Maintainer note | ~2017 | Mark Bates publishes "goth needs a new maintainer"; upkeep shifts community/gobuffalo-ward[^5]. |
| Go modules era | — | `go.mod` adopted; distributed as `github.com/markbates/goth` with `v1.x` tags. |
| Ongoing | 2026-02-11 | Still maintained via provider PRs on `master`; 50+ providers, most recent push[^2]. |

## References

[^1]: Goth README — "This package was inspired by omniauth." https://github.com/markbates/goth#readme
[^2]: GitHub API metadata for markbates/goth (created 2014-10-14, MIT, Go, ~6.6k stars, last push 2026-02-11). https://github.com/markbates/goth
[^3]: `Provider` and `Session` interface definitions. https://github.com/markbates/goth/blob/master/provider.go
[^4]: Goth README, "Security Notes" — default `gothic.Store` `CookieStore` options (`Secure: false`, `MaxAge: 86400*30`). https://github.com/markbates/goth#security-notes
[^5]: Mark Bates, "Goth Needs a New Maintainer" (linked as the repo homepage). https://blog.gobuffalo.io/goth-needs-a-new-maintainer-626cd47ca37b

## Tags

go, oauth, oauth2, openid-connect, authentication, social-login, sso, web, library, gorilla-sessions
