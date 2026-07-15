# varvet/pundit

> Authorization for Rails expressed as plain Ruby classes — no DSL, no runtime magic, just objects you write and call yourself.

[GitHub repo](https://github.com/varvet/pundit) ·
[API docs](https://www.rubydoc.info/gems/pundit) ·
[License: MIT](https://github.com/varvet/pundit/blob/main/LICENSE.txt)

## Overview

Pundit is a small authorization library for Ruby, almost always used with
Rails, maintained by Varvet (the Swedish consultancy formerly named Elabs)[^1].
It answers one question — "is this user allowed to do this to this record?" — by
having you write ordinary Ruby classes called policies. There is no permission
DSL, no rules table, no `can :manage, :all` grammar. A policy is a class whose
name matches a model plus the suffix `Policy`, whose constructor takes
`(user, record)`, and whose query methods (`update?`, `destroy?`) return a
boolean. Authorization checking is just calling those methods.

The defining tradeoff is verbosity in exchange for explicitness. Pundit gives
you almost nothing you could not write yourself — the maintainers say so in the
README ("There's no secret sauce here"). What it adds is conventions and a few
controller helpers (`authorize`, `policy_scope`, `verify_authorized`) so a team
writes authorization the same way in every action. This suits apps where the
logic is genuinely per-record; it scales badly the other way, where a
role-and-permission matrix becomes one nearly-identical policy method per action.
The library is mature and low-churn rather than evolving — the API has been
essentially fixed for years and the code base is a few hundred lines, which for a
security-adjacent dependency is a feature.

## Getting Started

```sh
bundle add pundit
```

Then `include Pundit::Authorization` in `ApplicationController`.

```ruby
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def update?
    user.admin? || !record.published?
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      user.admin? ? scope.all : scope.where(published: true)
    end
  end
end
```

```ruby
# app/controllers/posts_controller.rb
def update
  @post = Post.find(params[:id])
  authorize @post            # raises Pundit::NotAuthorizedError if update? is false
  @post.update(post_params) ? redirect_to(@post) : render(:edit)
end

def index
  @posts = policy_scope(Post)  # runs PostPolicy::Scope#resolve
end
```

`rails g pundit:install` generates an `ApplicationPolicy` base class with sane
defaults; `rails g pundit:policy post` scaffolds an individual policy.

## Architecture / How It Works

Pundit's whole mechanism is naming convention plus reflection. `authorize @post`
does three things: infers `PostPolicy` from the record's class, infers the query
method from the current controller action name (`update` → `update?`),
instantiates `PostPolicy.new(current_user, @post)`, and raises
`Pundit::NotAuthorizedError` unless the method returns truthy. `authorize`
returns the record it was given, so `@post = authorize Post.find(...)` reads
cleanly. Everything else is a variation: pass an explicit query symbol
(`authorize @post, :publish?`), pass `policy_class:` to override inference, or
pass a class instead of an instance when there is no specific record.

Scopes are the parallel mechanism for collections. `policy_scope(Post)` finds
`PostPolicy::Scope`, calls `.new(current_user, Post).resolve`, and returns
whatever it builds — usually a narrowed `ActiveRecord::Relation`. This is the
piece people underuse: record-level `authorize` protects `show`, but only a
policy scope keeps forbidden rows out of an `index` listing.

The user comes from `current_user` by default; override `pundit_user` when the
actor has a different name (Rails 8's authentication generator uses
`Current.user`). Namespaced policies (`authorize [:admin, post]` →
`Admin::PostPolicy`) let one model carry different rules per context. Two
development-only guards, `verify_authorized` and `verify_policy_scoped`, run in
`after_action` and raise if you forgot to call `authorize` / `policy_scope`; the
README is emphatic that these are forgetfulness aids, not a security backstop.
Coupling to Rails is loose — the core logic lives in `Pundit` module methods that
take a user explicitly, so it is usable outside controllers and, with effort,
outside Rails.

## Production Notes

- **The scope trap.** The most common real bug is authorizing single records
  with `authorize` but building collections with a raw model query instead of
  `policy_scope`, so forbidden rows leak into `index` listings silently.
  `verify_policy_scoped` in `after_action` for `index` actions is the defense.
- **`verify_authorized` is not enforcement.** It is trivially satisfiable — one
  `skip_authorization` call marks the action "authorized." Treat it as a linter,
  not a gate; real enforcement is that every branch actually calls `authorize`.
- **`nil` users.** A policy is instantiated with whatever `current_user`
  returns, including `nil`. Guest-accessible apps must handle `user.nil?` in every
  method, or raise in the `ApplicationPolicy` constructor for login-only apps;
  otherwise you get a `NoMethodError` on `nil` instead of a clean denial.
- **User caching.** Pundit memoizes the user context per request. When an app
  switches the acting user mid-request (impersonation, "log in as"), call
  `pundit_reset!` or stale permissions apply.
- **Verbosity at scale.** For RBAC-shaped apps a policy method per action per
  model becomes large and repetitive; teams mitigate with shared modules and
  inheritance. Pundit ships no role model — roles are your problem.
- **Testing / errors.** Policies are plain objects, unit-testable without
  controllers (`PostPolicy.new(user, post).update?`; `pundit-matchers` is the
  RSpec companion). `Pundit::NotAuthorizedError` carries `query`, `record`, and
  `policy`, which you `rescue_from` to render a 403 and, via I18n, per-action
  messages.

## When to Use / When Not

**Use when:**
- Authorization is genuinely per-record and depends on record state (ownership,
  publication status, workflow stage), not just a role name.
- You want authorization logic that is plain, testable Ruby your team can read
  without learning a DSL.
- You value a tiny, stable, low-churn dependency in a security-sensitive spot.

**Avoid when:**
- Your model is a clean role/permission matrix — a rules-based library expresses
  that with far less code.
- You want permissions defined/edited at runtime or stored in the database;
  Pundit policies are code, evaluated at request time.
- You are not in Ruby, or want an out-of-the-box role system; Pundit provides
  neither roles nor a persistence layer.

## Alternatives

- CanCanCan/cancancan — ability-DSL, all rules in one `Ability` class; terser for
  role-matrix apps, harder to test in isolation. Use instead when permissions are
  role-shaped and centralized.
- palkan/action_policy — policy-object model like Pundit with caching, explicit
  authorization context, and aliases built in; use instead when you want Pundit's
  shape with more batteries.
- RolifyCommunity/rolify — role management only, not authorization; pair with
  Pundit rather than replace it when you need DB-backed roles.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2012-11 | Started under `elabs/pundit`. |
| 1.0.0 | ~2015 | First stable release; policy + scope conventions settled. |
| 2.0.0 | ~2019 | Dropped old Ruby support; repo later moved to `varvet/pundit` as Elabs rebranded to Varvet[^1]. |
| 2.1–2.2 | 2019–2021 | Maintenance; scope and namespacing refinements. |
| 2.3.0 | ~2022 | `include Pundit::Authorization` introduced; bare `include Pundit` deprecated. |
| 2.4.0 | ~2024 | Ruby/Rails version bumps, minor fixes. |
| 2.5.0 | ~2025 | Latest line; `pundit_reset!` for user switching documented. |

Dates above are approximate; the changelog and RubyGems are authoritative.[^2]

## References

[^1]: Varvet AB (formerly Elabs), sponsor and maintainer. https://www.varvet.com
[^2]: Pundit changelog and release history. https://github.com/varvet/pundit/blob/main/CHANGELOG.md · https://rubygems.org/gems/pundit

## Tags

ruby, rails, authorization, access-control, policy-objects, security, plain-old-ruby, gem, rbac, web
