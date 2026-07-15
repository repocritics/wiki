# heartcombo/devise

> The default full-stack authentication engine for Rails — ten opt-in modules mounted on Warden.

[GitHub repo](https://github.com/heartcombo/devise) ·
[License: MIT](https://github.com/heartcombo/devise/blob/main/MIT-LICENSE)

## Overview

Devise is a Rack-based authentication solution for Ruby on Rails, built as a
Rails engine on top of the Warden middleware[^1]. It originated at Plataformatec
(the Brazilian consultancy where José Valim worked before creating Elixir) and is
now maintained under the `heartcombo` organization[^2]. For most of the last
decade it has been the default answer to "how do I add login to a Rails app,"
to the point that its conventions (`authenticate_user!`, `current_user`,
`devise_for`) are treated as part of Rails itself by much of the community.

Its defining design choice is modularity. Authentication is split into ten
concerns — Database Authenticatable, Registerable, Recoverable, Rememberable,
Trackable, Validatable, Confirmable, Lockable, Timeoutable, Omniauthable — that
you enable per model by listing them in a `devise` call. This makes the happy
path very short but pushes complexity into configuration and into the controller
override layer, which is where most real projects eventually end up living.

The recurring tension: Devise is excellent for session-cookie web apps and
deliberately awkward for anything else. It is a full-stack engine that ships
controllers, views, mailers, and routes — great when your app matches its shape,
frustrating when you want a JSON API, custom flows, or token auth, because you
are then fighting an opinionated engine rather than composing small pieces.

## Getting Started

Devise 5 targets Rails 7 and newer[^3].

```sh
bundle add devise
rails generate devise:install   # writes config/initializers/devise.rb (read it)
rails generate devise User      # adds modules to the model + routes + migration
rails db:migrate
```

```ruby
# app/models/user.rb — enable only the modules you need
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable
end

# app/controllers/dashboard_controller.rb
class DashboardController < ApplicationController
  before_action :authenticate_user!   # redirects to sign-in when logged out

  def show
    @user = current_user              # nil-safe helper Devise injects
  end
end
```

Restart the server (and stop `spring`) after changing Devise config — stale
route helpers and login failures from not restarting are a common first-hour
confusion[^4].

## Architecture / How It Works

Underneath the generators, Devise is a thin, opinionated layer over **Warden**,
a general Rack authentication framework[^1]. Warden owns the actual
"authenticate this request" logic through *strategies* and stores the
authenticated record in the Rack session; Devise supplies Rails-flavored
strategies, a `mapping` system for scopes (`:user`, `:admin`), the ten model
modules, and the MVC surface (controllers, views, mailers, routes) that Warden
itself does not provide.

Because it is a Rails **engine**, everything ships inside the gem: the sign-in
form, the password-reset mailer, the routes. You customize by *overriding* —
`rails generate devise:views` copies view templates into your app,
`rails generate devise:controllers` subclasses the engine controllers, and
`devise_for ..., controllers: { sessions: "users/sessions" }` rewires routing.
This inheritance-and-override model is powerful but means your customizations are
coupled to Devise's internal method names (`resource`, `resource_name`,
`after_sign_in_path_for`), which are semi-public API.

Password storage uses **bcrypt** via the Database Authenticatable module, with a
configurable cost (`stretches`). Historically Devise supported pluggable
encryptors; those were deprecated and extracted to the separate
`devise-encryptable` gem, and token-based authentication (`token_authenticatable`)
was removed in 3.1 over timing-attack and token-leak concerns[^5]. That removal
is why there is no built-in API token story today.

Scopes let multiple models be signed in simultaneously — a `User` and an `Admin`
can hold independent sessions in the same request. Each scope generates its own
helper set (`current_admin`, `authenticate_admin!`, `admin_signed_in?`).

## Production Notes

The differentiators between "works in the tutorial" and "works in production":

- **CSRF token rotation on sign-in.** Devise clears the CSRF token after
  authentication (`clean_up_csrf_token_on_authentication`). If your login is an
  AJAX/Turbo request and you do not perform a full navigation afterward,
  subsequent non-GET requests can fail with "Can't verify CSRF token
  authenticity" until the page reloads.
- **API-only mode is second-class.** Devise assumes cookies and a stored session.
  For token/JWT auth you need an add-on such as `devise-jwt`, and in
  `config.api_only` apps you must wire session and flash middleware back in.
  Treat "Devise for a pure JSON API" as a project, not a checkbox.
- **Password-reset tokens can leak into logs.** The README documents that reset
  tokens passed as query params may be captured by Rails request logging; filter
  them or rely on the tokened routes as designed[^6].
- **User enumeration.** By default, error messages and timing can reveal whether
  an email is registered. Enable `paranoid` mode and generic messages if that is
  in your threat model.
- **Confirmable is stateful and easy to misconfigure.** `reconfirmable` (require
  re-confirmation on email change) interacts with `confirm_within` and the
  unconfirmed-access grace window; getting these wrong locks users out or leaves
  accounts permanently unconfirmed.
- **Trackable stores IP addresses and timestamps** on the user row — a privacy /
  GDPR consideration that teams often enable without noticing.
- **No authorization.** Devise answers "who are you," never "what may you do."
  Pair it with Pundit or CanCanCan.

**Upgrade pain.** The 3.x → 4.x jump moved parameter sanitization from the model
to the controller (the `devise_parameter_sanitizer` API), so any app permitting
extra sign-up fields had to be rewritten[^7]. Encryptor removal in the same era
forced legacy password hashes onto `devise-encryptable`. Major versions track
Rails' supported range, so upgrading Devise is usually entangled with a Rails
upgrade.

## When to Use / When Not

**Use when:**
- You are building a server-rendered Rails app with cookie sessions.
- You want registration, confirmation email, password reset, lockout, and
  "remember me" without writing them.
- You need multiple independent login scopes (users vs. admins).
- Your customization needs are covered by overriding controllers and views.

**Avoid when:**
- You are shipping a pure JSON/token API — the built-in flow fights you; consider
  Rodauth or a hand-rolled `has_secure_password` setup.
- You want to understand and own every line of your auth (Devise's indirection is
  large; small apps may prefer clearance or sorcery).
- You are on your first Rails app — the maintainers themselves recommend building
  auth from scratch first to learn the framework[^8].

## Alternatives

- jeremyevans/rodauth — more security features (WebAuthn, OTP, audit logging) and
  a cleaner separation from the framework; use when auth is a first-class concern
  or you want a JSON API without add-ons.
- thoughtbot/clearance — minimal email/password only; use when you want something
  small and fully readable.
- Sorcery (Sorcery/sorcery) — provides the building blocks, you write the
  controllers; use when you want Devise's features without its engine indirection.
- omniauth/omniauth — OAuth/social login only; complements rather than replaces
  Devise (Devise's Omniauthable wraps it).
- Rails 8's built-in `authentication` generator — for new apps wanting a
  no-dependency starting point; use when your needs are basic and you would
  rather not add a gem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010 | First stable release; Warden-based engine, module system. |
| 2.0 | 2012-01 | Rails 3.1+, asset-pipeline era cleanups. |
| 3.0 | 2013-08 | Rails 4 / strong parameters; token auth removed in 3.1[^5]. |
| 4.0 | 2016-06 | Parameter sanitizer moved to the controller; bcrypt-only[^7]. |
| 4.9 | 2023 | Final 4.x line; broad Rails 5–7 support. |
| 5.0 | 2026 | Rails 7+ baseline; drops end-of-life Rails/Ruby versions[^3]. |
| 5.0.4 | 2026-05-08 | Latest release at time of writing[^9]. |

Still actively maintained: ~24.3k stars, ~5.5k forks, last pushed June 2026,
with a steady release cadence rather than large rewrites — a mature project whose
churn now tracks the Rails release calendar more than new features.

## References

[^1]: Warden — Rack authentication middleware Devise is built on. https://github.com/wardencommunity/warden
[^2]: heartcombo organization (maintainers of Devise, formerly Plataformatec). https://github.com/heartcombo
[^3]: Devise README, "Getting started" — "Devise 5 works with Rails 7 onwards." https://github.com/heartcombo/devise#getting-started
[^4]: Devise README, note on restarting the app after configuration changes. https://github.com/heartcombo/devise#getting-started
[^5]: Devise 3.1 release notes — removal of `token_authenticatable` and encryptor deprecation. https://github.com/heartcombo/devise/blob/main/CHANGELOG.md
[^6]: Devise README, "Password reset tokens and Rails logs." https://github.com/heartcombo/devise#password-reset-tokens-and-rails-logs
[^7]: Devise README, "Strong Parameters" — sanitization moved to controller in Devise 4. https://github.com/heartcombo/devise#strong-parameters
[^8]: Devise README, "Starting with Rails?" — recommends not using Devise for a first Rails app. https://github.com/heartcombo/devise#starting-with-rails
[^9]: Devise releases — v5.0.4, 2026-05-08. https://github.com/heartcombo/devise/releases

## Tags

ruby, rails, authentication, warden, session-management, rack, mvc-engine, bcrypt, web-security, oauth
