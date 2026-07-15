# kaminari/kaminari

> Scope-based, Rails-Engine-backed pagination for Ruby web apps — the long-standing alternative to will_paginate.

[GitHub repo](https://github.com/kaminari/kaminari) ·
[Wiki / recipes](https://github.com/kaminari/kaminari/wiki) ·
[License: MIT](https://github.com/kaminari/kaminari/blob/master/MIT-LICENSE)

## Overview

Kaminari is a pagination library for Ruby web applications, written by Akira
Matsuda (`amatsuda`) and first released in 2011[^1]. Its distinguishing design
choice is that pagination is expressed as ordinary query scopes — `User.page(7).per(50)`
returns a normal `ActiveRecord::Relation`, not a special collection class — so
paginator calls chain freely with other conditions before or after them. It
deliberately avoids monkey-patching `Array`, `Hash`, `Object`, or `ActiveRecord::Base`,
which was a direct reaction to the more invasive style of the older will_paginate.

The view layer is packaged as a mountable Rails Engine. The `paginate` helper
renders each link through a partial template shipped inside the gem, so the
markup can be overridden per app, per theme, or per A/B test by generating and
editing partials — without patching the helper. This Engine-plus-scope split is
the whole architecture, and it is why Kaminari is ORM- and template-agnostic:
the same core drives ActiveRecord, Mongoid, and MongoMapper adapters, and ERB,
Haml, or Slim templates.

Kaminari is mature rather than fast-moving. With ~8.7k stars and a codebase
untouched by rewrites since the 1.0 restructure, it is best read as
infrastructure that is done: the last pushes are maintenance and Rails-version
compatibility rather than new features[^2]. For a paginator, that stability is a
feature. The main tension is that its defaults assume classic offset pagination,
which does not scale to very large tables — a problem the library acknowledges
but only partially addresses.

## Getting Started

```ruby
# Gemfile
gem 'kaminari'
```

```ruby
# controller — page is a scope; add an explicit order to avoid surprises
@users = User.order(:name).page(params[:page]).per(25)
```

```erb
<%# view — renders ?page=N links wrapped in an HTML5 <nav> %>
<%= paginate @users %>
<%= page_entries_info @users %>
```

```sh
% rails g kaminari:config   # optional: default_per_page, max_per_page, window, etc.
% rails g kaminari:views default  # copy link partials into app/views/kaminari/ to customize
```

Pagination is 1-indexed: `page(0)` behaves like `page(1)`. Kaminari never adds
an `ORDER BY` on your behalf, so unordered paginated queries return
database-dependent row orders across pages.

## Architecture / How It Works

The `kaminari` gem is a meta-package over three components[^3]:

- **kaminari-core** — the scope logic (`page`, `per`, `padding`, page-number
  predicates) and the paginator/tag rendering model. ORM- and framework-neutral.
- **kaminari-activerecord** — the Active Record adapter that injects `page` into
  models via a concern.
- **kaminari-actionview** — the Action View adapter providing `paginate`,
  `link_to_next_page`, `page_entries_info`, and the partial templates.

`gem 'kaminari'` pulls the AR + Action View adapters, which reference core. Other
stacks swap an adapter: `kaminari-mongoid`, `kaminari-mongo_mapper`,
`kaminari-sinatra`, `kaminari-grape`. This is a genuinely clean separation — the
core has no knowledge of SQL or HTTP.

`per` is defined on the page scope, not the model, because it only makes sense
after `page`. Internally `per` sets `limit` and `page`/`padding` set `offset`, so
Kaminari scopes are just `LIMIT`/`OFFSET` sugar plus a `COUNT(*)` to compute
`total_pages`. That count query is the load-bearing cost of the whole design.

The paginate helper is a loop over page-number "tags" (first, prev, gap/truncate,
page, next, last) governed by `window`, `outer_window`, `left`, and `right`.
Each tag is a partial in the Engine, rendered through the Rails I18n API — labels
like `« First` live in a YAML file inside the gem and are overridable per locale.

## Production Notes

- **`COUNT(*)` is the scaling wall.** The default `paginate` helper needs the
  total row count to render "Last »" and the page window. On large or heavily
  filtered tables that count is often slower than fetching the page itself. Use
  `.without_count`[^4] to drop the count query entirely — you lose `total_pages`,
  `last_page?`, and the full helper, and get only `link_to_next_page` /
  `link_to_prev_page`. This is the standard escape hatch for big datasets, but it
  is opt-in and easy to forget until a table grows.
- **Deep offsets are slow regardless of Kaminari.** `LIMIT 25 OFFSET 1000000`
  makes the database scan and discard a million rows. Kaminari does not implement
  keyset/cursor pagination; for that you drop to a hand-rolled `WHERE id > ?`
  scheme or a different gem.
- **No implicit ordering.** Because Kaminari never injects `ORDER BY`, paginated
  results can silently repeat or skip rows between pages if the underlying query
  is unordered. Always order paginated queries on a stable key.
- **`max_paginates_per` / `max_pages` are the abuse guards.** Without them,
  `?per=100000` or `?page=99999999` is accepted as given; set model-level caps to
  bound query cost from untrusted params.
- **Ajax helper caveat.** `paginate @users, remote: true` emits `data-remote`
  attributes that depend on rails-ujs; the README notes this path works with
  Rails < 7.2, so Turbo-based apps should not rely on it[^5].
- **Custom templates are copies, not subclasses.** `rails g kaminari:views` copies
  partials into your app; they then diverge from the gem's and will not pick up
  upstream markup changes on upgrade. Re-diff after major version bumps.

## When to Use / When Not

**Use when:**
- You want offset pagination in a Rails/ActiveRecord app with near-zero setup.
- You need to restyle pagination markup or support multiple themes/locales.
- You target a non-Rails Ruby stack (Sinatra, Grape) or a non-AR ORM (Mongoid)
  and want one consistent paginator API across them.

**Avoid when:**
- You paginate very large tables where `COUNT(*)` or deep `OFFSET` dominates —
  reach for keyset/cursor pagination instead.
- You need infinite-scroll/cursor semantics as a first-class feature.
- You want an actively feature-developed library; Kaminari is in maintenance mode.

## Alternatives

- mislav/will_paginate — the older, more monkey-patch-heavy paginator; use it if
  you inherit a codebase already built around it.
- ddnexus/pagy — use when performance and memory matter; it avoids per-page
  object allocation and has first-class keyset support.
- Custom keyset pagination — use when tables are large enough that any
  `COUNT`/`OFFSET` approach is too slow; trade convenience for `WHERE id > ?`.
- kaminari-mongoid / kaminari-grape — same Kaminari core when your stack is
  Mongoid or Grape rather than AR + Action View.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2011 | First release by Akira Matsuda; scope + Engine design[^1]. |
| 1.0 | 2016 | Split into kaminari-core / -activerecord / -actionview[^3]. |
| 1.1 | 2017 | Bug fixes, newer Rails support. |
| 1.2 | 2019+ | Rails 6/7 compatibility, `max_pages`, `without_count` refinements. |

Exact patch dates vary by component; consult the CHANGELOG and RubyGems for the
precise release a given gem version shipped in.

## References

[^1]: Repository metadata, created 2011-02-06; MIT-LICENSE "Copyright (c) 2011- Akira Matsuda." https://github.com/kaminari/kaminari
[^2]: GitHub repo stats and last-push date (Feb 2026) as of fetch; ~8.7k stars, ~1.07k forks. https://github.com/kaminari/kaminari
[^3]: README, "Other Framework/Library Support" — the kaminari gem = kaminari-core + kaminari-activerecord + kaminari-actionview. https://github.com/kaminari/kaminari#other-frameworklibrary-support
[^4]: README, "Paginating Without Issuing SELECT COUNT Query" (`without_count`). https://github.com/kaminari/kaminari#paginating-without-issuing-select-count-query
[^5]: README, "Ajax Links (via rails-ujs, works with Rails < 7.2)." https://github.com/kaminari/kaminari

## Tags

ruby, rails, pagination, activerecord, rubygem, web, mongoid, view-helpers, offset-pagination, engine
