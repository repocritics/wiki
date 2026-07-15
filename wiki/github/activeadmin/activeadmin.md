# activeadmin/activeadmin

> A DSL for generating a full admin backend for a Rails application from your ActiveRecord models — CRUD, filtering, and dashboards with almost no view code.

[GitHub repo](https://github.com/activeadmin/activeadmin) ·
[Official website](https://activeadmin.info) ·
[License: MIT](https://github.com/activeadmin/activeadmin/blob/master/LICENSE)

## Overview

Active Admin is a Ruby on Rails engine, created by Greg Bell around 2010–2011, that generates administration interfaces from a declarative DSL[^1]. You call `ActiveAdmin.register Post` and get an index table, filters, form, show page, CSV/JSON export, and pagination — all driven by introspecting the model. It targets the common internal-tooling case: give non-technical staff a way to edit production data without building bespoke screens.

The defining tension is the classic scaffold-vs-app one. For the 80% case, Active Admin is close to zero-code and stays terse as models grow. The remaining 20% — a custom widget, an unusual workflow, a non-ActiveRecord data source — pushes you off the happy path into overriding controllers, writing Arbre view components, or dropping to partials, where the DSL's magic becomes an obstacle rather than a shortcut. Teams that treat it as "the admin panel forever" tend to fight it; teams that treat it as "a fast internal CRUD you may outgrow" get the most value.

It is one of the oldest and most-depended-on gems in the Rails ecosystem (nearly 9.7k stars, ~3.3k forks as of mid-2026), and it is still actively maintained, though development pace is deliberate rather than fast. The project stitches together several other gems — Formtastic (forms), Ransack (search/filtering), Kaminari (pagination), Arbre (Ruby-to-HTML view layer), and Devise (auth) — so its behavior and its security surface are largely inherited from those dependencies[^2].

## Getting Started

```ruby
# Gemfile
gem "activeadmin"
gem "devise" # Active Admin's default auth integration
```

```bash
bundle install
rails generate active_admin:install   # sets up initializer, AdminUser, routes
rails db:migrate
rails generate active_admin:resource Post
```

```ruby
# app/admin/posts.rb
ActiveAdmin.register Post do
  permit_params :title, :body, :published   # Rails strong params, required

  index do
    selectable_column
    id_column
    column :title
    column :published
    actions
  end

  filter :title
  filter :published

  form do |f|
    f.inputs do
      f.input :title
      f.input :body
      f.input :published
    end
    f.actions
  end
end
```

The admin lives at `/admin`. `permit_params` is mandatory for any writable attribute — omissions silently drop fields on save, a frequent first-time gotcha.

## Architecture / How It Works

Active Admin is a mounted Rails engine. The core moving parts:

- **Registration DSL** — `app/admin/*.rb` files are evaluated at boot. Each `register` block builds an `ActiveAdmin::Resource` that dynamically generates a `Rails` controller and routes. This means adding admin pages has a boot-time cost and that errors in an admin file can break app startup, not just the admin.
- **Arbre** — Active Admin's own Ruby DSL for building HTML as an object tree instead of ERB/Haml[^3]. `index`, `show`, and `form` blocks are Arbre. It is expressive for programmatic layout but is a second templating system to learn, and stack traces from it are harder to read than template errors.
- **Formtastic** — powers `form do |f| ... end` and the `f.input` semantics (type inference from column type/associations).
- **Ransack** — powers `filter` and the query-string search. Ransack is also the single biggest security consideration (see below).
- **Kaminari** — pagination on index pages.
- **Inherited Resources** — historically provided the controller CRUD superclass; the project has been reducing this coupling over major versions.

The controllers are real Rails controllers, so `controller do ... end` blocks, `before_action`, and `member_action`/`collection_action` give you full escape hatches. The rendering layer, not the controller layer, is where customization gets awkward.

**Asset-pipeline shift.** Historically the view layer shipped SCSS/SASS plus jQuery-era JavaScript through Sprockets. The 4.x line moves the styling to Tailwind CSS with Flowbite components[^4] — visible in the current dependency list — which is a substantial break in how you theme and override the UI. Which line you are on materially changes customization advice, so pin the major version when searching for help.

## Production Notes

- **Ransack allowlisting is now mandatory and will surprise you.** Ransack 4.0 (2022) stopped allowing search/sort on arbitrary attributes for security and requires each model to define `ransackable_attributes` / `ransackable_associations`[^5]. Upgrading Active Admin across that boundary breaks filters silently or raises until every admin-searchable model declares its allowlist. This is the most common upgrade-pain report.
- **Boot coupling.** Because `app/admin/*.rb` runs at load time, a bad reference (renamed model, missing column) can prevent the whole app from booting, not just the admin. Keep admin files honest with migrations.
- **N+1 queries on index pages.** Columns that traverse associations generate a query per row unless you set `includes` on the resource. Large tables with association columns degrade quickly; profile before shipping wide index views.
- **`permit_params` drift.** Adding a model column does not add it to the admin form's permitted params; the field appears but silently fails to persist until you update `permit_params`. Easy to miss in review.
- **Customization ceiling.** Anything beyond CRUD + filters — dashboards with charts, multi-step flows, bulk operations with side effects — is doable but means writing Arbre or partials and overriding controller actions. At that point you are maintaining an app inside a framework designed to avoid maintaining an app.
- **Not a public-facing UI.** It is an internal admin. It assumes trusted, authenticated operators; do not expose it to end users or treat its authorization as a substitute for real app-level policies (pair with Pundit/CanCanCan for per-record rules).
- **Security disclosure** goes through Tidelift's coordinated process, not GitHub issues[^2].

## When to Use / When Not

**Use when:**
- You have a Rails app with ActiveRecord models and need an internal CRUD admin quickly.
- Your admin needs are mostly standard: list, filter, edit, export, simple dashboards.
- Non-technical staff need to edit data and you want to spend near-zero front-end effort.

**Avoid when:**
- The admin is a core product surface with heavy custom UX — you will fight the DSL.
- Your data isn't ActiveRecord (Active Admin is tightly bound to AR introspection).
- You want a modern SPA/React admin or a framework-agnostic tool.
- You need fine-grained, per-record authorization as the primary concern rather than an add-on.

## Alternatives

- thoughtbot/administrate — lighter, generates plain ERB views you own and edit; less magic, more code, easier to customize.
- railsadminteam/rails_admin — comparable batteries-included Rails admin; more configuration-driven, different DSL tradeoffs.
- avo-hq/avo — modern, partly-commercial Rails admin with a Hotwire/Tailwind UI; use when you want a maintained current-stack alternative and can accept the licensing.
- forest-admin/forest-admin — hosted/external admin panel that connects to your DB; use when you want an admin without embedding it in the app.
- Custom Rails controllers + Pundit — use when the admin is a real product feature that will keep growing past CRUD.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2011 | First public releases; long-running 0.x/1.0.0.pre beta era[^1]. |
| 1.0.0 | 2017 | Long-awaited stable release after years of pre-releases. |
| 2.0 | 2019 | Dropped older Rails/Ruby, dependency modernization. |
| 3.0 | 2023 | Rails 7 support, further dependency updates. |
| 4.0 (dev) | 2024–2026 | Tailwind CSS + Flowbite view layer, moving off legacy SCSS/JS asset stack[^4]. |

## References

[^1]: Active Admin — created by Greg Bell; acknowledged in the project README. https://github.com/activeadmin/activeadmin#acknowledgements
[^2]: Active Admin README — dependencies (Arbre, Devise, Formtastic, Inherited Resources, Kaminari, Ransack) and Tidelift security-contact process. https://github.com/activeadmin/activeadmin/blob/master/README.md
[^3]: Arbre — an HTML views-in-Ruby DSL used by Active Admin. https://github.com/activeadmin/arbre
[^4]: Active Admin README dependency list includes Tailwind CSS and Flowbite, reflecting the 4.x view-layer direction. https://github.com/activeadmin/activeadmin/blob/master/README.md
[^5]: Ransack 4.0 authorization changes requiring `ransackable_attributes`/`ransackable_associations` allowlists. https://activerecord-hackery.github.io/ransack/going-further/other-notes/#authorization-allowlistingdenylist

## Tags

ruby, rails, admin-panel, cms, crud, dsl, activerecord, backend, internal-tooling, ransack
