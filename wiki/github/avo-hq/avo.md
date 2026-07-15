# avo-hq/avo

> A code-driven admin panel and internal-tool framework for Ruby on Rails, built on Hotwire.

[GitHub repo](https://github.com/avo-hq/avo) ·
[Official website](https://avohq.io) ·
[License: LGPL-3.0 (Community) / commercial (Pro, Advanced)](https://github.com/avo-hq/avo/blob/main/LICENSE.md)

## Overview

Avo generates a CRUD admin panel for a Rails application from Ruby class definitions. You describe each Active Record model as a `Resource` — its fields, associations, actions, filters, and authorization — and Avo mounts a full back-office UI (index tables, detail pages, forms, search, dashboards) as a Rails engine at `/avo`. It targets the same job as ActiveAdmin, RailsAdmin, and Administrate: give a team a maintainable internal tool without hand-writing controllers and views for every model.

Its defining technical choice is the frontend stack. Avo is built on Hotwire (Turbo + Stimulus) and Tailwind, not on a heavy JS SPA, so pages are server-rendered HTML with Turbo Frames for partial navigation and the client footprint stays small[^1]. That keeps Avo close to idiomatic Rails and avoids the asset-pipeline entanglement that older admin gems suffered.

The other defining fact is commercial. Avo is open-core: the Community tier is LGPL-3.0, but a large share of the framework's advanced surface — dashboards, dynamic filters, deeper resource tooling, and more — sits in Pro and Advanced tiers unlocked by a paid license key[^2]. Copyright is held by DEPENDEV SRL, and the project is run as a sustained commercial business by its author Adrian Marin rather than as a volunteer effort. That is the central tension: the Community edition is genuinely usable, but the docs and roadmap pull toward the paid tiers, and the line between "free" and "buy a license" is something you evaluate up front, not after adoption.

## Getting Started

```bash
# Gemfile
bundle add avo

# generate the initializer and mount the engine
bin/rails generate avo:install
```

```ruby
# app/avo/resources/user.rb  — declare a resource
class Avo::Resources::User < Avo::BaseResource
  self.title = :name
  self.search = { query: -> { query.ransack(name_cont: params[:q]).result } }

  def fields
    field :id, as: :id
    field :name, as: :text, required: true
    field :email, as: :text
    field :admin, as: :boolean
    field :posts, as: :has_many
  end
end
```

Visit `/avo` and the model is browsable, searchable, and editable. Generator names and file layout are version-specific; follow the installation guide for your major version[^3].

## Architecture / How It Works

Avo ships as a Rails engine. Mounting it wires up its own routes, controllers, and views under a namespace, so it lives beside your application rather than inside it — you do not modify your models or app controllers to adopt it.

- **Resources** are the core abstraction: one Ruby class per model (or per view of a model) declaring `fields`, and optionally `filters`, `actions`, `scopes`, and authorization. Fields are typed (`:text`, `:boolean`, `:belongs_to`, `:has_many`, `:file`, custom types), and one declaration drives index, show, and form rendering.
- **Rendering** is Hotwire-native. Pages are ERB/ViewComponent-rendered HTML; Turbo Frames handle in-place navigation and Stimulus controllers handle client interactivity. There is no separate API/JSON layer or client bundle to build.
- **Authorization** delegates to Pundit policies[^4]. Avo looks up a policy per resource and calls the standard `index?`, `show?`, `update?`, `destroy?` predicates, reusing whatever authorization you already have rather than inventing its own.
- **Extensibility** happens through custom fields, custom tools (arbitrary pages outside the CRUD model), resource tools, and actions. This is where an app grows from "admin panel" into "internal tool platform."

The tiered gems matter architecturally: features are gated by license, not by a plugin boundary you control. In Avo 3 and later the distribution consolidated around a single `avo` gem whose Pro/Advanced capabilities activate from a license key[^2], which simplifies dependency management but means the paywall lives inside the framework you depend on.

## Production Notes

- **The open-core boundary is the first thing to map.** Before building, confirm which features your admin needs sit in Community versus Pro/Advanced. Discovering mid-project that dashboards or dynamic filters require a license is the most common Avo surprise. Pricing is a per-app subscription and a real line item, not a one-time cost[^2].
- **Major-version upgrades are non-trivial.** The 2→3 transition changed resource file layout and class conventions (resource naming and namespacing) enough to require a migration pass; treat each major bump as a scheduled task and read the upgrade guide before bumping[^5]. Community and paid tiers move together, so you cannot pin one and float the other.
- **Search is only as good as the query you write.** Resource search runs whatever scope you give it; large tables want a real backend (Ransack over indexed columns, or an external engine) rather than naive `LIKE` scans.
- **It is coupled to Active Record.** Avo assumes Active Record models. Non-AR data sources, service objects, or heavily denormalized schemas fit awkwardly and push you toward custom tools rather than resources.
- **UI customization has a ceiling.** Because rendering is Avo's, deep visual divergence from the built-in Tailwind design means overriding views/components — more effort than the DSL, and prone to breaking on upgrades. Avo is strong for standard back-office UIs and weaker when the admin is really a customer-facing, bespoke product surface.

## When to Use / When Not

**Use when:**
- You have a Rails + Active Record app and want a maintainable internal admin without hand-rolling controllers and views.
- You are comfortable in the Hotwire/Tailwind world and want the admin to stay server-rendered and close to Rails.
- You already use Pundit (or will), and want authorization reuse.
- A commercial license and vendor-backed support are acceptable — or even preferred over a purely volunteer project.

**Avoid when:**
- You need every feature free and unrestricted; the open-core split will eventually route you to a paid tier.
- Your data isn't Active Record, or the admin is really a bespoke user-facing product rather than a back office.
- You want a fully community-governed project with no single commercial owner.

## Alternatives

- activeadmin/activeadmin — the incumbent; Arbre-DSL admin, fully MIT, larger community. Use when you want a free, battle-tested admin and don't mind its older stack.
- thoughtbot/administrate — deliberately minimal, view-based and Rails-native. Use when you want to own and edit the generated views directly.
- railsadminteam/rails_admin — auto-generates an admin from your models with little config. Use for a quick zero-DSL admin over a conventional schema.
- palkan/motor-admin — low-code admin configured through a DB-backed UI. Use when non-developers should build views without writing Ruby.
- forestadmin/forest — SaaS-hosted admin layer. Use when you want a managed, UI-configured back office and accept an external service.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-04-26 | Initial development by Adrian Marin / DEPENDEV SRL[^6]. |
| 1.0 | 2021 | First stable major; Hotwire-based resource DSL. |
| 2.x | 2022 | Expanded fields, actions, filters; open-core tiers established. |
| 3.x | 2023–2024 | Consolidated gem + license-key model; resource layout changes[^5]. |
| 4.x | current (2026) | Current major; docs and installation target 4.0[^3]. |

Release dates for the 1.0–4.0 majors are approximate; verify against the changelog before citing a specific day.

## References

[^1]: Avo README — "Powered by Hotwire" and feature list. https://github.com/avo-hq/avo
[^2]: Avo pricing and licensing (Community LGPL-3.0; Pro/Advanced commercial license keys). https://avohq.io/pricing
[^3]: Avo installation guide (4.0). https://docs.avohq.io/4.0/installation.html
[^4]: Avo authorization via Pundit. https://docs.avohq.io/4.0/authorization.html
[^5]: Avo upgrade guide. https://docs.avohq.io/4.0/upgrade.html
[^6]: GitHub repository metadata, avo-hq/avo (created 2020-04-26; copyright DEPENDEV SRL). https://github.com/avo-hq/avo
[^7]: Avo LICENSE.md — LGPL-3.0 for the Community edition; commercial license for Pro/Advanced. https://github.com/avo-hq/avo/blob/main/LICENSE.md

## Tags

ruby, ruby-on-rails, rails, admin-panel, internal-tools, cms, crud, hotwire, open-core, low-code, pundit
