# ViewComponent/view_component

> Object-oriented view layer for Rails: a Ruby class plus a template, rendered and tested in isolation.

[GitHub repo](https://github.com/ViewComponent/view_component) ·
[Official website](https://viewcomponent.org) ·
[License: MIT](https://github.com/ViewComponent/view_component/blob/main/MIT-LICENSE.txt)

## Overview

ViewComponent is a framework for building reusable, testable, encapsulated view components in Ruby on Rails. It was extracted from GitHub's own monolith and first released publicly in 2019, originally under the name `actionview-component`[^1]. Each component is a plain Ruby object (a subclass of `ViewComponent::Base`) paired with a sidecar template file; the class holds logic and the template holds markup, replacing the ad-hoc mix of partials, helpers, and instance variables that Rails views accumulate over time.

The core value proposition is not new capability but discipline. Rails partials render against an implicit, mutable binding — any instance variable or helper in scope is reachable — which makes them cheap to write and hard to reason about at scale. ViewComponent forces an explicit interface: data enters through `initialize`, the template can only see what the object exposes, and the whole thing can be unit-tested with `render_inline` in milliseconds without booting a controller. That trade — more ceremony per component in exchange for isolation and testability — is the framework's defining tension, and the reason it fits large teams better than small apps.

ViewComponent is community-governed under its own GitHub organization and is deliberately *not* part of Rails core[^2]. Rails' primitives (ActionView, partials) remain the default; ViewComponent is an opt-in layer that most heavily benefits design-system and component-library work.

## Getting Started

```ruby
# Gemfile
gem "view_component"
```

```ruby
# app/components/message_component.rb
class MessageComponent < ViewComponent::Base
  def initialize(name:)
    @name = name
  end
end
```

```erb
<%# app/components/message_component.html.erb %>
<h1>Hello, <%= @name %>!</h1>
```

```erb
<%# render from any view %>
<%= render(MessageComponent.new(name: "World")) %>
```

```ruby
# test/components/message_component_test.rb
require "test_helper"

class MessageComponentTest < ViewComponent::TestCase
  def test_renders
    render_inline(MessageComponent.new(name: "World"))
    assert_text("Hello, World!")
  end
end
```

## Architecture / How It Works

A ViewComponent is compiled, not interpreted per request. On first render (in production, once at boot via eager loading) the template is compiled into a real Ruby method — `call` — on the component class. Subsequent renders invoke that method directly, which is why ViewComponent is measured as faster than equivalent partials: it skips ActionView's partial lookup and rendering machinery on the hot path[^3]. Template resolution is by convention — `MessageComponent` finds `message_component.html.erb` next to it — and multiple template engines (ERB, Haml, Slim) are supported through ActionView's handlers.

Key subsystems:

- **Slots** (`renders_one` / `renders_many`) — the mechanism for passing blocks of markup into a component, analogous to named yields. Slots were rewritten between v1 and v2 ("Slots v2"), and the older API was removed in v3, which is the single biggest migration hazard in the project's history.
- **Collections** — `render(RowComponent.with_collection(@rows))` renders one instance per element with an optional `_counter` and `_iteration`, avoiding a Ruby-level loop in the caller.
- **Previews** — `ViewComponent::Preview` renders components in isolation in the browser, the same idea as Storybook. This is what the `lookbook` gem builds on for a richer UI.
- **Sidecar assets** — a component can live in its own directory alongside its template and (via community add-ons) its JS/CSS, giving a self-contained unit.

Because a component is just an object, everything Ruby offers applies: inheritance, composition, mixins, `content_tag`, and constructor validation. The framework leans on ActionView for rendering context (helpers, routes, `content_for`) rather than reimplementing it, so components still run inside Rails' view binding and can reach `helpers.*` — a pragmatic escape hatch that also partially undermines the isolation the framework sells.

## Production Notes

- **Eager loading and boot cost.** In production every component template compiles at boot. A large component library measurably increases boot time; this is usually acceptable but noticeable in apps with thousands of components. In development, templates recompile on change, and stale-compilation confusion (edits not appearing) almost always traces back to caching or a missed reload.
- **The `helpers` escape hatch is a smell magnet.** Reaching `helpers.current_user` or global state from inside a component reintroduces exactly the implicit coupling ViewComponent exists to remove. Codebases that lean on it heavily get the ceremony cost without the isolation benefit. Passing dependencies through `initialize` is the intended path.
- **Testing speed is the real payoff.** `render_inline` tests run without a controller or full request cycle, so a component suite is fast enough to run on every save. Teams that adopt ViewComponent primarily for testability report the biggest wins; teams that adopt it for "cleaner views" alone often find the boilerplate not worth it on small apps.
- **Migration between majors is not free.** v2 renamed the gem and namespace; v3 removed the deprecated Slots v1 API and dropped older Ruby/Rails support; v4 continued API cleanup. The project maintains parallel release lines (v3.x backports still shipped in 2026 alongside v4.x), so pinning a major and reading the upgrade guide before bumping is mandatory[^4].
- **Not a JS component model.** ViewComponent renders HTML on the server. It composes well with Hotwire/Turbo/Stimulus but does not manage client-side state; pairing it with a heavy SPA is redundant.

## When to Use / When Not

**Use when:**
- You are building a design system or shared component library across a large Rails app or multiple teams.
- View-layer testability matters and controller/integration tests are too slow or too coarse.
- Partials have grown into unmaintainable webs of implicit instance variables and helpers.
- You want server-rendered HTML with a Hotwire/Turbo front end.

**Avoid when:**
- The app is small and partials are working fine — the per-component boilerplate is real overhead.
- You prefer writing markup as Ruby with no template files, in which case Phlex is a better philosophical fit.
- Your rendering is client-side; a server component layer adds little.

## Alternatives

- phlex-ruby/phlex — views as pure Ruby methods, no template files; the main modern rival, faster and more composable but a bigger departure from Rails idioms.
- Rails partials (built into rails/rails) — zero dependencies; use when isolation and unit testing are not worth the ceremony.
- trailblazer/cells — the older view-component concept for Rails; predates ViewComponent, less active today.
- ViewComponent/view_component_contrib — companion patterns and structure (namespacing, sidecar CSS/JS) layered on top of ViewComponent itself.
- komposable/komponent — component-oriented Rails workflow built around webpack/Vue-era tooling; largely superseded.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2019-08-20 | First public release; extracted from GitHub's monolith, originally `actionview-component`[^1]. |
| 2.0.0 | 2020-03-23 | Renamed gem/namespace to ViewComponent; Slots v2 introduced. |
| 3.0.0 | 2023-04-24 | Removed deprecated Slots v1 and legacy APIs; dropped older Ruby/Rails[^4]. |
| 4.0.0 | 2025-07-30 | Continued API cleanup; new major line. |
| 4.12.0 | 2026-06-04 | Recent v4 release; v3.x still receives backports in parallel. |

## References

[^1]: Joel Hawksley / GitHub Engineering, "Encapsulating Ruby on Rails views" — origin and extraction of ViewComponent from GitHub's monolith. https://github.blog/2020-12-15-encapsulating-ruby-on-rails-views/
[^2]: ViewComponent governance and organization. https://viewcomponent.org/
[^3]: ViewComponent documentation, "Frequently asked questions — performance / benchmarks." https://viewcomponent.org/known_issues.html
[^4]: ViewComponent changelog and upgrade notes. https://github.com/ViewComponent/view_component/blob/main/docs/CHANGELOG.md

## Tags

ruby, rails, view-layer, components, server-side-rendering, design-system, testing, html, encapsulation, hotwire
