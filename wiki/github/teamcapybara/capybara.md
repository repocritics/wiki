# teamcapybara/capybara

> Ruby acceptance-test DSL that drives a web app the way a user would, with automatic waiting and a swappable browser backend.

[GitHub repo](https://github.com/teamcapybara/capybara) ·
[Official website](http://teamcapybara.github.io/capybara/) ·
[License: MIT](https://github.com/teamcapybara/capybara/blob/master/LICENSE.txt)

## Overview

Capybara is a high-level acceptance/integration testing library for Ruby web
applications, created by Jonas Nicklas and first released in 2009[^1]. It sits
above a browser driver and exposes a DSL that reads like a description of user
behavior — `visit`, `fill_in`, `click_button`, `expect(page).to have_content`
— rather than raw WebDriver calls. It is the substrate underneath Rails system
tests (Rails 5.1+) and the default choice for feature/acceptance specs in the
RSpec, Cucumber, and Minitest ecosystems.

Two design decisions define Capybara. First, **driver agnosticism**: the same
test can run against the pure-Ruby `:rack_test` driver (fast, in-process, no
JavaScript) or a real browser via `:selenium`, with no changes to the test
body — you flip a flag or tag an example `js: true`. Second, **automatic
synchronization**: finders and matchers retry against the DOM until they
succeed or a timeout elapses (`default_max_wait_time`, 2 seconds by default),
so tests against Ajax-heavy pages do not need hand-written `sleep` calls. This
waiting model is the single most important thing to understand about
Capybara; nearly every subtle bug and every subtle feature traces back to it.

At ~10.2k stars with active commits through mid-2026, Capybara is mature and
maintained rather than fast-moving. The core DSL has been stable for a decade;
recent churn is mostly Selenium/CDP compatibility and driver deprecations.

## Getting Started

Requires Ruby 3.0.0 or later. Add to your `Gemfile`:

```ruby
gem 'capybara'
```

```ruby
# spec_helper.rb (RSpec + Rails)
require 'capybara/rspec'

RSpec.describe 'the signin process', type: :feature do
  it 'signs me in' do
    visit '/sessions/new'
    within('#session') do
      fill_in 'Email',    with: 'user@example.com'
      fill_in 'Password', with: 'password'
    end
    click_button 'Sign in'
    expect(page).to have_content 'Success'   # waits up to 2s for the text
  end
end
```

Tag an example `js: true` (or a Cucumber scenario `@javascript`) to switch that
test to `Capybara.javascript_driver` — `:selenium` (Firefox) by default;
`:selenium_chrome_headless` is also pre-registered.

## Architecture / How It Works

The public surface is a set of mixins on `Capybara::Session` and
`Capybara::Node::Element`, split by concern:

- **`Node::Finders`** — `find`, `all`, `find_field`, `find_button`. These are
  the retrying primitives; they re-run the query until an element appears (or
  the ambiguity/absence condition is resolved) within the wait window.
- **`Node::Matchers`** — `has_selector?`, `has_content?`, and the RSpec
  `have_*` matchers. Positive matchers wait for presence; **negative matchers
  wait for absence** (`have_no_content` blocks until the text is gone). This
  asymmetry is why `expect(page).to have_no_x` is correct and
  `expect(page).not_to have_x` is a race.
- **`Node::Actions`** — `click_link`, `fill_in`, `select`, `attach_file`.
  Actions locate-then-act, inheriting the same waiting.

Element location goes through the **selector system**. Capybara compiles its
semantic selectors (`:field`, `:link`, `:button`, `:fillable_field`) down to
XPath via the `xpath` gem, then hands the query to the driver. You can register
custom selectors and add filters to existing ones.

**Drivers** implement a narrow interface (visit, find, evaluate JS, manage
cookies/windows). `:rack_test` talks directly to a Rack app in-process — no
server, no JS, no external URLs. Selenium and CDP drivers boot a real server
(Capybara starts one on an ephemeral port, Puma by default) and drive a real
browser. Because the browser runs in a separate thread/process, it does not
share the test's database transaction — the source of Capybara's most famous
gotcha (below).

## Production Notes

**The database-transaction trap.** With a JS driver, the app server runs in a
separate thread. If your suite wraps each test in a transaction (the RSpec/Rails
default), the browser's requests hit the DB on a different connection and cannot
see uncommitted rows — tests fail with "record not found" that look like race
conditions. The historical fix is `database_cleaner` in truncation/deletion
mode for JS tests; modern Rails/Selenium can instead share a single connection
across threads, but this is version- and adapter-sensitive.

**Flakiness is almost always a waiting bug, not Capybara.** Common causes:
using `find(...).click` then asserting on stale references; asserting absence
with `not_to have_*` (races) instead of `to have_no_*`; reading `current_path`
directly instead of the waiting `have_current_path` matcher; or mixing
non-waiting Capybara calls (`all`, `first`, direct `.text`) with async DOM
updates. Raising `default_max_wait_time` masks symptoms and slows the suite.

**Ambiguous matches.** Under the default `Capybara.match = :smart` /
`exact: false`, a locator that resolves to more than one element raises
`Capybara::Ambiguous`. This is intentional (it catches under-specified tests)
but surprises people migrating from tools that silently pick the first hit.

**Visibility.** By default Capybara only sees *visible* elements — a real user
can't click `display:none`. `find(..., visible: :all)` opts out. Searches are
also **case-sensitive** because they lower to XPath.

**Driver churn is the real maintenance cost.** The DSL is stable; the drivers
underneath are not. `capybara-webkit` (QtWebKit) and `poltergeist` (PhantomJS)
were the standard headless drivers for years and are now dead as their engines
were abandoned. Current headless options are Selenium (Chrome/Firefox headless)
and CDP-based drivers like `cuprite`/`apparition`. Selenium major bumps
(3→4) periodically require capability-config changes in registered drivers.

**Upgrade note:** Capybara 3.0 (2018) normalized whitespace in text matching
(`have_content` collapses/trims whitespace) and changed default matching
behavior — a real breaking change that broke brittle assertions on exact
spacing.

## When to Use / When Not

**Use when:**
- You test a Ruby (Rack/Rails/Sinatra) web app end-to-end and want tests
  written in user language, not WebDriver calls.
- You want the same specs to run headless-fast (`rack_test`) and in a real
  browser (`selenium`) by flipping a tag.
- You are on Rails system tests — you are already using Capybara.

**Avoid when:**
- Your app is not driven from Ruby — Playwright or Cypress are better native
  fits for JS/TS stacks.
- You need fine-grained control over network interception, tracing, or
  multi-tab automation — raw Playwright/CDP exposes more than Capybara's
  driver interface abstracts.
- You are unit-testing view logic; Capybara is acceptance-level and slower.

## Alternatives

- microsoft/playwright — use instead when your stack is JS/TS or you need
  network interception, video/trace, and modern auto-waiting out of the box.
- cypress-io/cypress — use for JS/TS front-end teams wanting an integrated
  runner, time-travel debugging, and browser-native execution.
- rubycdp/cuprite — use as a Capybara *driver* (not a replacement) when you
  want Chrome-only headless via CDP without a Selenium/WebDriver dependency.
- SeleniumHQ/selenium — use directly when you want raw WebDriver control and
  are willing to hand-write waiting and locators.
- watir/watir — use for a Ruby browser-automation API when you are not doing
  Rack/Rails acceptance testing and don't need Capybara's DSL.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2009-11 | Initial release by Jonas Nicklas[^1]. |
| 1.0 | 2012-01 | First stable line; established the driver + waiting model. |
| 2.0 | 2012-11 | API cleanup; stricter finders and matcher semantics. |
| 3.0 | 2018-05 | Whitespace normalization in text matching; default-match changes[^2]. |
| 3.40 | 2024 | Selenium 4 compatibility, Ruby 3.x support, ongoing maintenance[^3]. |

## References

[^1]: Capybara changelog and release history. https://github.com/teamcapybara/capybara/blob/master/History.md
[^2]: Capybara 3.0 upgrade notes (whitespace/text matching changes). https://github.com/teamcapybara/capybara/blob/master/History.md
[^3]: teamcapybara/capybara README and rubygems releases. https://rubygems.org/gems/capybara/versions

## Tags

ruby, testing, acceptance-testing, integration-testing, end-to-end, browser-automation, selenium, rack-test, rspec, cucumber, capybara, web-testing
