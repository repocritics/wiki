# thoughtbot/shoulda-matchers

> One-liner RSpec and Minitest matchers that assert Rails model, controller, and routing configuration by introspecting and exercising the framework.

[GitHub repo](https://github.com/thoughtbot/shoulda-matchers) ·
[Official website](https://matchers.shoulda.io) ·
[License: MIT](https://github.com/thoughtbot/shoulda-matchers/blob/main/MIT-LICENSE)

## Overview

Shoulda Matchers is a testing library from thoughtbot that provides declarative one-liner matchers for the "framework-y" parts of a Rails application — associations, validations, controller callbacks, strong parameters, routes, and enum/serialize/attachment macros. A line like `it { should validate_uniqueness_of(:email).scoped_to(:account_id) }` replaces a dozen lines of hand-written setup and assertion. It grew out of thoughtbot's original `shoulda` gem, which was split into `shoulda-context` (the `should`/`context` macros) and `shoulda-matchers` (the matchers themselves) in the early 2010s[^1].

The library's defining tension is stated openly in its own README: most matchers are implemented with Rails reflections and other introspection, so they test *configuration* rather than *behavior*, which cuts against the usual "test behavior, not implementation" advice[^2]. thoughtbot's position is that this is exactly the point — declarative Rails macros (`belongs_to`, `validates_uniqueness_of`, `before_action`) are precisely the code that is easy to get subtly wrong and tedious to test by hand, so a tool that pins their presence and options is worth the coupling. For genuine business logic, the maintainers recommend writing ordinary behavior specs instead.

As of 2026 it is a mature, still-actively-maintained gem (last pushed July 2026) with ~3.6k stars and hundreds of millions of downloads, and is close to a default dependency in RSpec-tested Rails codebases. Its health is best read not from stars but from how quickly it tracks Rails: because it reaches into Rails internals, each Rails release tends to be met with a matching shoulda-matchers release.

## Getting Started

```ruby
# Gemfile
group :test do
  gem 'shoulda-matchers', '~> 8.0'
end
```

```ruby
# spec/rails_helper.rb (RSpec + Rails)
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

```ruby
# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  describe 'associations' do
    it { should belong_to(:account) }
    it { should have_many(:posts).dependent(:destroy) }
  end

  describe 'validations' do
    subject { build(:user) }            # provide a *valid* subject
    it { should validate_presence_of(:email) }
    it { should validate_uniqueness_of(:email).case_insensitive }
  end
end
```

For Minitest, swap `with.test_framework :minitest` and use the `should` macro from the umbrella `shoulda` gem. Non-Rails projects that use ActiveModel/ActiveRecord standalone integrate `with.library :active_record`/`:active_model` and mix the matcher modules into the relevant example groups manually.

## Architecture / How It Works

Matchers fall into families keyed to the Rails layer they inspect: `ActiveModel` (validations, `has_secure_password`), `ActiveRecord` (associations, `enum`, `serialize`, `normalize`, `encrypt`, ActiveStorage attachments, db columns/indexes), `ActionController` (`permit`, `redirect_to`, `render_template`, callback matchers), routing (`route`), and independent matchers (`delegate_method`). When integrated with `with.library :rails`, the gem auto-includes each family into the matching RSpec example-group type — ActiveRecord/ActiveModel matchers only in `type: :model`, controller matchers only in `type: :controller`, `route` only in `type: :routing`[^2].

Under the hood a matcher does two things. First it *introspects*: it reads Rails' own metadata — `reflect_on_association`, the validators registered on a class, the enum definitions, callback chains — to confirm a macro was declared with the expected options. Second, for many matchers, it *exercises* the model: it takes the `subject`, mutates an attribute to a boundary value, runs the real validation, and inspects `errors`. This second half is why `validate_uniqueness_of` is different from every other validation matcher: to prove uniqueness it must persist (or find) a conflicting record and hit the database, so it needs a real, valid subject and a live connection.

This design makes the matchers concise but tightly coupled to Rails' non-public internals. That coupling is the whole story of the project's maintenance: reflection APIs and validator internals shift between Rails versions, so shoulda-matchers is effectively version-locked to Rails and ships releases in lockstep with it. The current line targets Ruby 3.3+, Rails 7.2+, RSpec 3.x, and Minitest 5.x[^3].

## Production Notes

- **`validate_uniqueness_of` is the classic footgun.** It requires a persisted or buildable valid record and executes real DB queries; on a fresh subject with a non-null-constrained schema it will raise or emit guidance asking you to set a `subject { create(:thing) }`. It is also measurably slower than pure-introspection matchers because of the round trips — heavy use in large suites shows up in profiles.
- **Always give validation specs a valid subject.** `subject { build(:model) }` (FactoryBot or a plain valid instance) prevents unrelated validations from failing the matcher. "When in doubt, provide an instance of the class under test" is thoughtbot's own rule[^2].
- **Controller matchers are on the wrong side of an RSpec trend.** `render_template`, `redirect_to`, `set_flash`, and the callback matchers live in `type: :controller` specs, which rspec-rails has de-emphasized in favor of request specs; one-liner controller matchers do not carry over cleanly to `type: :request`. New apps leaning on request specs get less value from the ActionController family.
- **Upgrade shoulda-matchers with Rails, not after.** Because matchers read Rails internals, a Rails major/minor bump can surface matcher failures until you move to the shoulda-matchers release that supports it. Pin ranges deliberately; the README documents which last version supports each older Ruby/Rails floor (v3.1.3, v4.5.1, v6.5.0, v7.0.1)[^3].
- **Boolean and enum columns bite.** `validate_inclusion_of` on a boolean column and `validate_presence_of` on associations/`has_secure_password` have well-known quirks; the matchers usually detect these and print a specific hint rather than silently passing — read the failure message, it is unusually instructive.
- **It is a test-only dependency.** Keep it in `group :test`; it has no place in `development`/`production` and adds no runtime weight to the app.

## When to Use / When Not

**Use when:**
- You maintain a Rails app with RSpec (or Minitest via `shoulda`) and want to pin association options, validations, strong params, enums, and callbacks concisely.
- You are testing the declarative, easy-to-misconfigure macro surface of Rails rather than bespoke domain logic.
- You want failure messages that spell out exactly which option (`dependent`, `scoped_to`, `class_name`) diverged.

**Avoid when:**
- You are testing real business behavior and outcomes — write ordinary behavior specs; introspection matchers couple tests to implementation.
- You are building request-spec-first controllers and would rely mainly on the ActionController family.
- You are outside the supported Ruby/Rails matrix and cannot pin to a legacy line, or you want to avoid a dependency that must move in lockstep with Rails.

## Alternatives

- rspec/rspec-rails — the underlying framework; hand-write `expect(model).to be_valid` style behavior specs when you want to test outcomes rather than configuration.
- thoughtbot/shoulda — umbrella gem bundling matchers plus context macros; use it when you are on Minitest and want the `should`/`context` DSL too.
- thoughtbot/shoulda-context — the `context`/`should` block macros alone, without the matchers, for Test::Unit/Minitest structure.
- teamcapybara/capybara — different layer; use it when you should be asserting user-visible behavior end to end instead of model configuration.
- minitest/minitest — plain assertions plus Rails' built-in `ActiveSupport::TestCase` helpers when you prefer no matcher DSL at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | ~2011–2012 | Extracted from the original `shoulda` gem; matchers split from `shoulda-context`[^1]. |
| 3.0 | ~2016 | Significant internal rework of the matcher engine and messaging. |
| 4.0 | ~2019 | Dropped older Ruby/Rails; v3.1.3 kept for Ruby < 2.4 / Rails < 4.1[^3]. |
| 5.0 | ~2021 | Newer Rails support; v4.5.1 kept for Ruby < 3.0 / Rails < 6.1[^3]. |
| 6.0 | ~2023 | Rails 7 era matchers; v6.5.0 is last supporting Rails < 7.1[^3]. |
| 7.0 | ~2024 | v7.0.1 kept for Ruby < 3.3 / Rails < 7.2[^3]. |
| 8.0 | current | Targets Ruby 3.3+, Rails 7.2+, RSpec 3.x, Minitest 5.x[^3]. |

## References

[^1]: thoughtbot, "Shoulda Matchers" — project README, history of the `shoulda` split into `shoulda-context` and `shoulda-matchers`. https://github.com/thoughtbot/shoulda-matchers
[^2]: README — "A Note on Testing Style" and "Availability of RSpec matchers in example groups": matchers use reflections/introspection and are auto-included by example-group type. https://github.com/thoughtbot/shoulda-matchers#a-note-on-testing-style
[^3]: README — "Compatibility": supported Ruby 3.3+/Rails 7.2+/RSpec 3.x/Minitest 5.x and the legacy version floors (v3.1.3, v4.5.1, v6.5.0, v7.0.1). https://github.com/thoughtbot/shoulda-matchers#compatibility

## Tags

ruby, rails, rspec, minitest, testing, matchers, activerecord, activemodel, test-tooling, thoughtbot
