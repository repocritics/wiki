# activerecord-hackery/ransack

> Object-based search for ActiveRecord — turn a `params[:q]` hash into a scoped, sortable query without writing SQL.

[GitHub repo](https://github.com/activerecord-hackery/ransack) ·
[Official website](https://activerecord-hackery.github.io/ransack/) ·
[License: MIT](https://github.com/activerecord-hackery/ransack/blob/main/MIT-LICENSE)

## Overview

Ransack builds ActiveRecord queries from a nested parameter hash. A form submits fields named like `q[name_cont]` or `q[price_gteq]`; Ransack parses the attribute (`name`), the predicate (`_cont`, `_gteq`), and the value, then assembles the corresponding `WHERE`/`ORDER BY` via Arel[^1]. The result is a normal `ActiveRecord::Relation`, so it chains with pagination, includes, and further scopes. It ships no runtime dependencies beyond Rails itself and requires no separate search server — the entire feature set is standard Ruby, ERB, and SQL.

It descends from Ernie Miller's earlier `meta_search`/`meta_where` gems and has been the default answer for "add a filter form to a Rails admin/index page" for over a decade[^2]. The design tension is inherent: Ransack's value is that it maps user-supplied strings directly onto database columns and predicates, and that same mechanism is a mass-assignment-style attack surface. If a user can name any column and any predicate, they can read columns you never meant to expose and construct expensive queries. Ransack 4.0 (2023) confronted this head-on by making attribute and association allowlists mandatory[^3] — a hard breaking change that split the ecosystem into pre-4.0 and post-4.0 apps.

Ransack is best for structured filtering (equality, ranges, `LIKE`, association joins, boolean flags) over a bounded set of columns. It is not a relevance-ranked, typo-tolerant, full-text engine and does not try to be.

## Getting Started

```ruby
# Gemfile — supported on Rails 8.1 / 8.0 / 7.2, Ruby 3.1+
gem 'ransack'
```

```ruby
# app/controllers/products_controller.rb
def index
  @q = Product.ransack(params[:q])
  @products = @q.result(distinct: true)
end
```

Since 4.0 you MUST allowlist what is searchable, or every query raises/returns nothing:

```ruby
class Product < ApplicationRecord
  belongs_to :category

  def self.ransackable_attributes(auth_object = nil)
    %w[name price created_at]
  end

  def self.ransackable_associations(auth_object = nil)
    %w[category]
  end
end
```

```erb
<%= search_form_for @q do |f| %>
  <%= f.label :name_cont, "Name contains" %>
  <%= f.search_field :name_cont %>
  <%= f.label :price_gteq, "Min price" %>
  <%= f.number_field :price_gteq %>
  <%= f.submit %>
<% end %>

<%= sort_link(@q, :price, "Price") %>   <%# toggles ORDER BY price asc/desc %>
```

Association predicates chain by name: `q[category_name_cont]` filters on the joined `categories.name`.

## Architecture / How It Works

`Model.ransack(params)` returns a `Ransack::Search` object, not a relation. The `Search` holds a tree of `Condition`s (attribute + predicate + values) and `Sort`s. Calling `.result` compiles that tree into Arel nodes and hands back an `ActiveRecord::Relation`[^1]. Nothing touches the database until you enumerate the relation, so Ransack composes with `.includes`, `.page`, and additional `where` clauses.

**Predicates** are the core vocabulary: `eq`, `not_eq`, `cont`/`not_cont` (`ILIKE %v%`), `start`/`end`, `matches`, `gt`/`lt`/`gteq`/`lteq`, `in`/`not_in`, `null`/`not_null`, `present`/`blank`, `true`/`false`, plus `_any`/`_all` suffixes for multi-value forms. Custom predicates register via `Ransack.configure`.

**Associations** are resolved by name-splitting: `articles_title_cont` walks `Model.reflect_on_association(:articles)`, joins it, and applies `title_cont` on the joined table. Because joins can multiply rows, `result(distinct: true)` is the common (and easily forgotten) guard.

**Ransackers** let you search on computed SQL expressions Ransack cannot infer from a column — you write raw Arel in a `ransacker :full_name do ... end` block. This is the escape hatch for concatenations, casts, and function calls, and it is also where you can accidentally reintroduce SQL injection if you interpolate user input instead of using Arel nodes.

**The allowlist layer** (`ransackable_attributes`, `ransackable_associations`, `ransackable_scopes`, `ransortable_attributes`) is consulted at compile time. Anything not returned by these class methods is silently dropped from the search rather than queried. The `auth_object` argument lets you return different allowlists per current user, enabling per-role search surfaces.

## Production Notes

**The 4.0 allowlist migration is the defining upgrade pain.** Before 4.0, all columns and associations were searchable by default; after 4.0, unlisted attributes are ignored[^3]. Apps that upgraded without defining `ransackable_attributes`/`ransackable_associations` on every searched model saw filters and sorts silently stop working — no exception, just empty or unfiltered results. Budget time to audit every model that appears in a search form or `sort_link`. Returning `super` (which defaults to column names) re-opens the surface and defeats the purpose; enumerate explicitly.

**Association searches join, and joins are expensive.** A deep predicate like `q[orders_line_items_product_name_cont]` generates multi-table joins with `ILIKE` and no index awareness. On large tables this is a full scan. Ransack does nothing to stop a user from submitting arbitrarily deep or costly filter combinations; rate-limit, cap allowlisted associations, and index the columns you expose.

**`distinct: true` vs pagination.** `result(distinct: true)` adds `SELECT DISTINCT`, which can conflict with `ORDER BY` on columns not in the select list (Postgres rejects this) and interacts awkwardly with some paginators. The alternative is a subquery/`EXISTS`-style scope, which Ransack does not build for you.

**`cont` is `LIKE`/`ILIKE`, not full-text.** Leading-wildcard `%term%` patterns cannot use a standard B-tree index, so `_cont` filters scan. For real search (ranking, stemming, typo tolerance) reach for Postgres `pg_search` or an external engine — Ransack is for filtering, not searching in the information-retrieval sense.

**Scopes must be opted in separately.** Named scopes are only ransackable if listed by `ransackable_scopes`, and by default scope values are passed through as-is, so validate scope arguments.

**Sorting is a separate allowlist.** `ransortable_attributes` defaults to `ransackable_attributes`, but if you want sortable and filterable sets to differ you override it independently. `sort_link` builds `q[s]=column+dir` params.

## When to Use / When Not

**Use when:**
- You need admin/index filter forms over known columns with equality, ranges, `LIKE`, and joins.
- You want search state to live in the URL (`?q[...]`) and round-trip through Rails form helpers.
- You want zero extra infrastructure — no Elasticsearch, no separate index to keep in sync.
- Per-role search surfaces via `auth_object` allowlists fit your authorization model.

**Avoid when:**
- You need relevance ranking, stemming, fuzzy/typo tolerance, or faceted full-text search.
- The search surface is untrusted and high-volume — arbitrary predicate/join construction is a DoS vector even with allowlists.
- Your filtering is a fixed handful of scopes; explicit `where`/named scopes are simpler and safer than a generic query builder.
- You are on a non-ActiveRecord ORM — Ransack is ActiveRecord-only.

## Alternatives

- Casecommons/pg_search — use instead when you need PostgreSQL full-text search (`tsvector`, ranking, trigram similarity) rather than column filtering.
- ankane/searchkick — use when you want Elasticsearch/OpenSearch-backed relevance, typo tolerance, and faceting, and can run the extra infrastructure.
- jhund/filterrific — use when you prefer wiring filters to explicit named scopes rather than auto-exposing attributes and predicates.
- bogdan/datagrid — use when you want a declarative grid (columns + filters + export) as a single object rather than form helpers over a relation.
- plataformatec/has_scope — use for the simplest case: mapping a few known params to a few known scopes, no query-builder abstraction.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011 | Created by Ernie Miller, successor to meta_search/meta_where[^2]. |
| 1.0 | 2013 | First 1.x line under the activerecord-hackery org. |
| 2.0 | 2018 | Rails 5+ support, predicate and configuration changes. |
| 3.0 | 2021 | Dropped older Rails/Ruby; internal Arel updates. |
| 4.0 | 2023-02 | Mandatory `ransackable_attributes`/`ransackable_associations` allowlists[^3]. |
| 4.x | 2024–2026 | Rails 7.2 / 8.0 / 8.1 and Ruby 3.1+ support; last push 2026-05[^4]. |

## References

[^1]: Ransack documentation — searching, predicates, and Arel-backed results. https://activerecord-hackery.github.io/ransack/
[^2]: Ransack README, "Contributors" — created by Ernie Miller, maintained by Sean Carroll, Deivid Rodriguez, Greg Molnar. https://github.com/activerecord-hackery/ransack
[^3]: Ransack upgrade notes on required allowlisting introduced in 4.0. https://activerecord-hackery.github.io/ransack/going-further/other-notes/#authorization-allowlistingdenylisting
[^4]: GitHub repository metadata (stars, forks, last push) — activerecord-hackery/ransack. https://github.com/activerecord-hackery/ransack

## Tags

ruby, ruby-on-rails, activerecord, sql, search, filtering, query-builder, rails-gem, backend, orm
