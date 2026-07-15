# ddnexus/pagy

> Pagination for Ruby and Rails that computes offsets with a single small object per request instead of decorating the collection.

[GitHub repo](https://github.com/ddnexus/pagy) ·
[Official website](https://ddnexus.github.io/pagy/) ·
[License: MIT](https://github.com/ddnexus/pagy/blob/master/LICENSE.txt)

## Overview

Pagy is a Ruby pagination library written by Domizio Demichelis (`ddnexus`), first published to RubyGems around 2018[^created]. Its pitch has always been resource cost: where Kaminari and WillPaginate mix pagination methods into ActiveRecord scopes and instantiate relatively heavy objects, Pagy computes the page math (offset, limit, page series) into one plain-Ruby `Pagy` instance and renders navigation HTML by string interpolation rather than through a template engine. The maintainer publishes a benchmark suite claiming roughly an order-of-magnitude advantage in speed and allocations over the incumbents[^bench]; the numbers are self-reported but the architectural reason for them — no collection decoration, no ERB per nav bar — is real and checkable in the source.

The library is "agnostic": the core knows nothing about ActiveRecord, Rails, or any specific ORM. You hand it a collection (or just a count) and it returns metadata plus the paginated slice. Rails, Sinatra, Hanami, Padrino, Sequel, arrays, Elasticsearch, Meilisearch, Typesense, and Searchkick are all supported through optional modules rather than a hard dependency.

The defining tension is manual wiring versus magic. Kaminari gives you `Model.page(params[:page])` as a scope and hides the plumbing; Pagy asks you to call `pagy(...)` in the controller and render helpers in the view, in exchange for lower overhead and no monkey-patching of your models. Historically this made Pagy the "fast but more assembly required" option. Version 43 — a full redesign released in 2026 — is explicitly aimed at closing that ergonomics gap, cutting configuration requirements by a claimed 99% and unifying the API behind a single `pagy(:technique, ...)` call[^v43].

## Getting Started

```ruby
# Gemfile
gem "pagy"
```

```ruby
# app/controllers/application_controller.rb
include Pagy::Method

# offset-based pagination (the classic LIMIT/OFFSET query)
@pagy, @records = pagy(:offset, Product.all)
```

```erb
<%# app/views/products/index.html.erb %>
<% @records.each do |product| %>
  <%= product.name %>
<% end %>

<%== @pagy.series_nav %>          <%# default pagy nav bar %>
<%== @pagy.series_nav(:bootstrap) %>  <%# or a framework style %>
<%== @pagy.info_tag %>            <%# "Displaying items 1-20 of 350" %>
```

## Architecture / How It Works

Pagy separates cleanly into a **backend** (page arithmetic) and a **frontend** (HTML rendering), and this split is why it stays cheap.

- **Backend.** A paginator such as `:offset`, `:countless`, `:keyset`, `:keynav`, `:calendar`, or one of the search-server variants takes your collection and produces a `Pagy` object holding `page`, `limit`, `offset`, `count`, and the computed page `series`. It never wraps or subclasses your collection — the records you get back are ordinary results of a `LIMIT`/`OFFSET` (or keyset) query the paginator issued.
- **Frontend.** Navigation helpers (`series_nav`, `series_nav_js`, `input_nav_js`, `info_tag`) build markup by interpolating a small string template. There is no partial rendering and no view-layer object graph per link, which is where the allocation savings come from.

Not every technique pays the `COUNT(*)` tax. `:offset` runs a count query to know the total pages; `:countless` deliberately skips it (fetching one extra row to detect whether a "next" page exists), and `:keyset`/`:keynav` use cursor/keyset pagination that stays fast on deep pages where `OFFSET N` degrades. Version 43 also adds a `:countish` paginator positioned as faster than plain offset while still supporting the full navigation UI[^v43].

Optional capabilities live in a toolbox of modules (formerly called "extras") — CSS-framework styles (Bootstrap, Bulma), JSON:API query-string handling, JavaScript-driven navigation, calendar/time-range pagination, and the search-engine adapters. As of v43 these are autoloaded on first use, so a method you never call consumes no memory. Pagy also ships its own minimal internationalization (`Pagy::I18n`) rather than pulling in the full `i18n` gem, keeping the dependency footprint at zero for the common case.

## Production Notes

- **Offset pagination degrades on deep pages.** This is a database property, not a Pagy bug: `OFFSET 100000` forces the engine to scan and discard rows. If users page far into large tables, move to `:keyset`/`:keynav`. If the count query itself is the bottleneck (large tables, expensive `WHERE`), `:countless` removes it at the cost of not knowing the total page count.
- **The JavaScript helpers require wiring.** The `*_js` navigation variants (`series_nav_js`, `input_nav_js`) depend on Pagy's `pagy.mjs`/`pagy.js` being present and initialized in the browser. Forgetting to sync or include that file is a recurring "the nav bar doesn't render/respond" support thread. Server-rendered helpers (`series_nav`) have no such requirement.
- **Out-of-range pages.** Requesting a page beyond the last one raises `Pagy::OverflowError` unless you configure overflow handling (e.g. clamp to the last page or return empty). Decide this explicitly; an unhandled overflow from a crawler hitting `?page=99999` is a common 500.
- **Major upgrades are real migrations.** Pagy has broken API across majors before, and v43 is described by the author as "a complete redesign… usage and API included." Do not treat a jump to 43 as a routine `bundle update`; read the upgrade guide, and budget for controller/view changes if you are coming from the v3–v9 era. The maintainer provides a dedicated migration guide from WillPaginate and Kaminari as well.
- **Counting large sets.** Because `:offset` needs a total, cache or approximate the count on very large tables if the count query shows up in your slow logs — or switch techniques rather than paginating with offset at all.

## When to Use / When Not

**Use when:**
- Pagination overhead (memory, allocations, per-request objects) matters at your traffic level.
- You want ORM/framework independence — arrays, Sequel, Rails, Sinatra, Hanami, or a search engine behind one API.
- You need techniques beyond offset: countless, keyset/cursor, or calendar/time-range pagination.
- You are fine wiring a controller call and view helpers explicitly.

**Avoid when:**
- You specifically want model-scope ergonomics (`Model.page(n)`) and Rails-style magic with minimal controller code — Kaminari fits that expectation better.
- Your app paginates trivial datasets where neither speed nor memory is a concern and the incumbent is already integrated; the migration cost may not repay.
- You need heavy view-template customization of nav markup through partials rather than string templates.

## Alternatives

- kaminari/kaminari — scope-based, Rails-idiomatic, view partials for nav; use it when developer ergonomics and `Model.page(n)` matter more than per-request cost.
- mislav/will_paginate — the original Rails paginator; use it when you already depend on it and have no performance pressure to migrate.
- basecamp/geared_pagination — cursor-friendly pagination with variable page sizes; use it when you want increasing page sizes or a Basecamp-style approach.
- rails/rails — ActiveRecord has `limit`/`offset` built in; use them directly when you need one query and no navigation UI at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2017-12 | Repo created; Pagy starts as a single-file, dependency-free gem.[^created] |
| 3.x | ~2019–2020 | Mature backend/frontend split and "extras"; the "snappy" baseline v43's notes benchmark against.[^bench] |
| 43 | 2026 | "Leap" release — full redesign, unified `pagy(:technique, …)` API, autoloaded toolbox, config reduced ~99%, new `:countish` and `:keynav` paginators.[^v43] |

## References

[^created]: GitHub API, `repos/ddnexus/pagy` — repository created 2017-12-31; last push 2026-07-09; 4,975 stars, 445 forks, MIT license. https://github.com/ddnexus/pagy
[^bench]: Pagy pagination comparison benchmarks (maintainer-published, self-reported). https://ddnexus.github.io/pagination-comparison/gems.html
[^v43]: Pagy README, "Version 43" release notes, and Upgrade Guide. https://ddnexus.github.io/pagy/guides/upgrade-guide/

## Tags

ruby, rails, pagination, gem, rubygems, keyset-pagination, cursor-pagination, activerecord, sinatra, hanami, performance, backend
