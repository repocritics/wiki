# FactoryBoy/factory_boy

> Declarative test-object factories for Python — the port of Ruby's factory_bot that replaces static fixtures with per-test object builders.

[GitHub repo](https://github.com/FactoryBoy/factory_boy) ·
[Documentation](https://factoryboy.readthedocs.io/) ·
[License: MIT](https://github.com/FactoryBoy/factory_boy/blob/master/LICENSE)

## Overview

factory_boy is a fixtures-replacement library for Python tests. Instead of maintaining static fixture files (Django's `loaddata` JSON, hand-written setup dicts) that drift out of sync with the models they seed, you declare a `Factory` subclass per model and instantiate exactly the object each test needs, overriding only the test-relevant fields. It is a direct descendant of thoughtbot's `factory_bot` (the Ruby library formerly named factory_girl)[^1], and the API mirrors that lineage.

The library was originally written by Mark Sandstrom and has been maintained for most of its life by Raphaël Barrois (rbarrois)[^2]. Its reach is broad because the core is ORM-agnostic: the base `factory.Factory` knows how to compute attribute values and call a constructor, and thin subclasses (`DjangoModelFactory`, `SQLAlchemyModelFactory`, `MongoEngineFactory`, `MogoFactory`) teach it how to persist. In practice the overwhelming majority of usage is Django, and much of the ecosystem's tribal knowledge (transaction handling, `get_or_create`, sequence resets) is Django-specific.

The defining tension is **explicitness versus boilerplate**. factory_boy requires you to declare every non-default field, which makes factories readable and intention-revealing but means a wide model needs a wide factory. Its main competitor, model_bakery, takes the opposite stance — introspect the model and auto-fill fields — trading readability for terseness. factory_boy chooses explicit declarations, and most of its power (SubFactory graphs, traits, post-generation hooks) exists to make that explicitness composable rather than repetitive.

## Getting Started

```bash
pip install factory_boy      # pulls in Faker as a hard dependency
```

```python
import factory
from myapp import models

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = models.User

    first_name = factory.Faker("first_name")
    last_name = factory.Faker("last_name")
    email = factory.LazyAttribute(lambda o: f"{o.first_name}.{o.last_name}@example.com".lower())
    is_active = True

class PostFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = models.Post

    title = factory.Sequence(lambda n: f"Post {n}")
    author = factory.SubFactory(UserFactory)   # cascades the create/build strategy
```

```python
user = UserFactory()                      # create(): saved to the DB
draft = PostFactory.build()               # build(): unsaved instance (author also unsaved)
vips = UserFactory.create_batch(5, is_active=True)
post = PostFactory(author__last_name="Doe")   # __ traverses into the SubFactory
```

## Architecture / How It Works

A `Factory` subclass is assembled by a metaclass (`FactoryMetaClass`) that collects two kinds of class attributes: **declarations** (anything that is a `factory.declarations.*` instance) and **plain values** (evaluated once, at class-definition time). Configuration lives in the inner `class Meta`, which the metaclass converts into a `FactoryOptions` object exposed as `_meta`.

At call time the factory runs a **build → generate** pipeline. It first resolves declarations in dependency order into a "step builder" / lazy stub, so that a `LazyAttribute` can read the value another declaration produced in the same instance. Only then does it hand the resolved kwargs to a strategy:

- **`build`** — construct the instance in memory, do not persist.
- **`create`** — construct and persist (the default when you call the factory directly).
- **`stub`** — return a plain attribute bag, never touching the model class.

The declaration classes are the real vocabulary. `LazyAttribute` and `LazyFunction` defer computation to instantiation time (the former receives the object under construction, the latter takes no argument). `Sequence` / `sequence` produce monotonically increasing unique values. `SubFactory` embeds another factory and — critically — **propagates the parent's strategy**, so `PostFactory.build()` builds an unsaved author while `PostFactory()` saves one. `RelatedFactory` and `post_generation` hooks run *after* the object exists, which is how you attach reverse-FK children or M2M rows. `Trait` bundles a named set of overrides toggled by a boolean. `SelfAttribute` and the `factory.` container references let declarations read sideways and upward through the object graph.

ORM subclasses override a small surface — chiefly `_create` and `_get_or_create` — to replace "call the constructor" with "call `Model.objects.create()`" and to honor `Meta.django_get_or_create`. Random data is delegated wholesale to the Faker library; factory_boy owns only the seeding contract (`factory.random.reseed_random`) so that a shared seed makes a run reproducible.

## Production Notes

**Sequences are class-state, not test-state.** A `Sequence` counter lives on the factory class and is *not* reset between tests. Tests that assert on exact sequence output (`"Post 0"`) pass in isolation and fail in a different run order. Either call `factory.reset_sequence()` in setup or, better, never assert on generated values — treat them as opaque.

**`create()` writes to the database, so transaction scoping matters.** Under Django you must run factories inside `TestCase` (transactional rollback) or the pytest-django `db`/`transactional_db` fixtures; a factory call in an unscoped context leaves real rows behind. SQLAlchemy factories need `Meta.sqlalchemy_session` (and a session-persistence policy) wired correctly or created objects silently never commit.

**SubFactory graphs cascade saves and dominate test time.** Every `SubFactory` on a `create` path is another INSERT, and deep graphs multiply quickly (a Post → Author → Organization chain saves three rows per post). The standard mitigations: use `.build()` when persistence is irrelevant, set `Meta.django_get_or_create` to deduplicate shared parents, pass an already-built instance instead of letting the SubFactory make a new one, and reach for `create_batch` sparingly — it is a Python loop of individual saves, not a `bulk_create`.

**Faker does not guarantee uniqueness.** `factory.Faker("email")` will collide often enough to trip a `unique` constraint when batching. Use `Sequence` (or Faker's `unique` proxy where supported) for genuinely unique columns; do not rely on entropy.

**`build()` skips validation.** In Django, building does not call `full_clean()`, so a `build()`-only test can pass on data that a real save would reject. If your invariants live in model validation rather than DB constraints, build-strategy tests give false confidence.

**post_generation extraction is a common footgun.** A `@post_generation` hook receives `create` and `extracted`; forgetting to guard on `if not create` or mishandling the extracted argument produces hooks that fire during `build` unexpectedly, or M2M assignments that raise because the object was never saved.

**Upgrade note:** the 3.0 line dropped Python 2 and made Faker a required dependency[^3]; projects pinning older 2.x factory_boy for Python 2 or optional-Faker reasons cannot take that jump without also touching their fixtures.

## When to Use / When Not

**Use when:**
- Your test objects are non-trivial graphs (foreign keys, nested relations) and you want each test to declare only what it cares about.
- You want the same factory to serve unit tests (`build`) and integration tests (`create`) via one definition.
- You need reproducible-but-realistic data and are already comfortable with Faker.
- You maintain multiple ORMs or a non-ORM plain-object codebase and want one fixtures tool across them.

**Avoid when:**
- You want zero-declaration setup and are Django-only — model_bakery introspects fields for you.
- Your objects are flat and few — plain pytest fixtures or literal constructors are less machinery.
- You only need random values, not object construction — depend on Faker directly.
- Your test data must be a fixed, byte-stable corpus (golden files) — static fixtures are the honest tool there.

## Alternatives

- model-bakery/model-bakery — Django-only; auto-fills fields by model introspection. Use instead when you want minimal boilerplate and don't need explicit, readable factory definitions.
- joke2k/faker — the random-data generator factory_boy sits on top of. Use directly when you need fake values but not object graphs or persistence.
- klen/mixer — multi-ORM auto-fill fixtures (Django, SQLAlchemy, Peewee, Mongo). Use when you want introspection-based generation beyond Django.
- pytest-dev/pytest — plain fixtures. Use when object setup is simple enough that a factory layer is overhead rather than leverage.
- Django `loaddata` fixtures — static JSON/YAML seed data. Use when you need a fixed, version-controlled dataset rather than per-test synthesis.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2011 | Early public releases; port of Ruby factory_girl semantics[^1]. |
| 2.0 | 2013 | API consolidation around declarations and strategies. |
| 2.9 | 2017 | Last of the 2.x line broadly used on Python 2. |
| 3.0 | 2020 | Dropped Python 2; Faker became a required dependency[^3]. |
| 3.1 | 2020 | Post-3.0 fixes and declaration refinements. |
| 3.2 | 2021 | Continued ORM and Faker-integration updates. |
| 3.3 | 2024 | Newer Python/Django support-matrix updates[^2]. |

## References

[^1]: thoughtbot/factory_bot (formerly factory_girl), the Ruby library factory_boy is modeled on. https://github.com/thoughtbot/factory_bot
[^2]: factory_boy changelog and documentation. https://factoryboy.readthedocs.io/en/stable/changelog.html
[^3]: factory_boy 3.0.0 release notes — Python 2 dropped, Faker made a hard dependency. https://factoryboy.readthedocs.io/en/stable/changelog.html

## Tags

python, testing, fixtures, factory, test-data, django, sqlalchemy, orm, faker, mock-data, pytest
