# fastapi/sqlmodel

> A thin layer that fuses Pydantic and SQLAlchemy so one Python class is both a validation model and a database table.

[GitHub repo](https://github.com/fastapi/sqlmodel) ·
[Official website](https://sqlmodel.tiangolo.com) ·
[License: MIT](https://github.com/fastapi/sqlmodel/blob/main/LICENSE)

## Overview

SQLModel is a Python ORM-adjacent library created by Sebastián Ramírez (tiangolo), the author of FastAPI, and first released in 2021[^1]. Its single idea: a class that inherits from `SQLModel` is simultaneously a Pydantic model (validation, serialization, JSON Schema) and a SQLAlchemy ORM model (tables, sessions, queries). The pitch is eliminating the duplicate "one Pydantic schema for the API, one SQLAlchemy model for the DB" boilerplate that FastAPI apps accumulate. The repo now sits under the `fastapi` GitHub organization (redirected from the original `tiangolo/sqlmodel`).

SQLModel is deliberately not a new ORM. It is described by its own docs as a thin layer on top of Pydantic and SQLAlchemy, and it inherits both their power and their seams. You still write SQLAlchemy `select()` statements, still open SQLAlchemy `Session`s, still reason about the SQLAlchemy Core/ORM distinction underneath. What SQLModel adds is the type-annotation-driven model definition and the Pydantic integration; everything else is delegated downward.

The defining tension is that "one class for everything" is clean for CRUD demos but leaks quickly. Real apps end up defining a small family of related classes (`HeroBase`, `Hero` with `table=True`, `HeroCreate`, `HeroPublic`, `HeroUpdate`) to separate what the API accepts, what the DB stores, and what the API returns — which is most of the duplication the library set out to remove, reorganized rather than eliminated. SQLModel is at its best for read-mostly, straightforward schemas and its most awkward at the boundaries where Pydantic validation semantics and SQLAlchemy persistence semantics disagree.

## Getting Started

```bash
pip install sqlmodel
# pulls in pydantic and sqlalchemy as dependencies
```

```python
from sqlmodel import Field, Session, SQLModel, create_engine, select


class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
    secret_name: str
    age: int | None = None


engine = create_engine("sqlite:///database.db")
SQLModel.metadata.create_all(engine)

with Session(engine) as session:
    session.add(Hero(name="Deadpond", secret_name="Dive Wilson"))
    session.commit()

    statement = select(Hero).where(Hero.name == "Deadpond")
    hero = session.exec(statement).first()
    print(hero)
```

Note `session.exec()` — SQLModel's typed wrapper around SQLAlchemy's `session.execute()` that returns model instances directly instead of `Row` tuples.

## Architecture / How It Works

SQLModel's core is metaclass work. `SQLModel` subclasses Pydantic's model machinery and, when a class is declared with `table=True`, also registers it with SQLAlchemy's declarative system and its shared `MetaData`. A field declared as `id: int | None = Field(default=None, primary_key=True)` is read both as a Pydantic field (for validation/serialization) and as a SQLAlchemy `Column` (for DDL and querying). The `Field()` function accepts both Pydantic constraints and SQLAlchemy column arguments (`primary_key`, `foreign_key`, `index`, `sa_column`) in one call.

Key mechanics:

- **`table=True` vs not.** A class without `table=True` is a pure Pydantic model (used for request/response schemas); with it, the class becomes a mapped table. The two share field definitions through inheritance but behave very differently — validation is notably relaxed on table models by default.
- **Relationships.** `Relationship(back_populates=...)` from SQLModel wraps SQLAlchemy's `relationship()`. Foreign keys are declared via `Field(foreign_key="hero.id")`. This is standard SQLAlchemy ORM relationship loading underneath, with all the lazy-loading and `selectinload` concerns intact.
- **`sa_column` escape hatch.** For anything SQLModel's `Field()` doesn't surface (server defaults, custom column types, composite constraints), you pass a fully-formed SQLAlchemy `Column` via `sa_column=`. This is the standard route once you outgrow the sugar.
- **Metadata.** `SQLModel.metadata` is the shared SQLAlchemy `MetaData` object; `create_all()`/`drop_all()` and Alembic autogeneration operate on it.

The important architectural fact is that SQLModel owns almost none of the runtime behavior. Query execution, connection pooling, transaction handling, relationship loading, and dialect specifics are all SQLAlchemy. Validation and serialization are all Pydantic. SQLModel is the definition-time glue, which is why its own codebase is small relative to what it appears to do — and why debugging frequently means reading SQLAlchemy or Pydantic docs, not SQLModel's.

## Production Notes

**Version pinning matters more than usual.** SQLModel has spent its entire life on `0.0.x` version numbers and does not promise a stable API. It also sits on top of two fast-moving dependencies. The single largest historical pain point: after Pydantic v2 shipped (mid-2023), SQLModel remained on Pydantic v1 for months before adding v2 support in the 0.0.14 release around the end of 2023[^2]. Teams that upgraded Pydantic independently hit hard breakage. Pin SQLModel, Pydantic, and SQLAlchemy together and upgrade them as a set.

**Validation is not enforced on table models the way newcomers expect.** With `table=True`, Pydantic validators do not run on assignment or on instance construction by default the same way they do on plain models — table instances are treated more like SQLAlchemy rows. Do input validation on a non-table `*Create` model, then construct the table model from the validated data (`Hero.model_validate(hero_create)`). Relying on the table class to validate untrusted input is a common bug.

**Migrations are not included.** SQLModel gives you `metadata.create_all()` for a first schema, not migrations. Production use means Alembic. Alembic autogenerate works but requires `import sqlmodel` in the migration environment and in generated scripts (columns render as `sqlmodel.sql.sqltypes.AutoString` and similar), and autogenerated diffs still need human review as with any SQLAlchemy project.

**Async requires dropping to SQLAlchemy pieces.** There is no distinct SQLModel async API; you use SQLAlchemy's `create_async_engine` and `AsyncSession` directly, with SQLModel models. `AsyncSession.exec()` availability and typing have been rougher than the sync path; many codebases fall back to `session.execute()` under async.

**The `str` / `AutoString` default.** Untyped `str` fields map to an unbounded string column (`AutoString`), which some databases treat as `TEXT` rather than a length-limited `VARCHAR`. If your dialect or DBA expects `VARCHAR(n)`, set it explicitly with `Field(sa_column=Column(String(255)))`.

**Maintenance is centralized.** The project is effectively driven by one primary author plus the FastAPI org's maintainers; issue and PR throughput has been uneven over the years, with long quiet stretches followed by bursts. It is widely used but should not be assumed to have same-week responsiveness to bugs.

## When to Use / When Not

**Use when:**
- You are building a FastAPI app and want request/response schemas and DB models to share definitions.
- Your schema is straightforward CRUD and you value the reduced boilerplate and editor autocompletion.
- You are comfortable reading SQLAlchemy docs when you hit the edges (you will).

**Avoid when:**
- You need advanced or exotic SQLAlchemy features front-and-center — use SQLAlchemy directly and add Pydantic schemas separately; the indirection buys you nothing.
- You want a mature, 1.0-stable, API-frozen dependency with predictable release cadence.
- You are not on FastAPI/Pydantic — the main payoff (schema sharing with FastAPI) mostly evaporates.
- You need heavy async ORM work today; the async ergonomics lag the sync path.

## Alternatives

- sqlalchemy/sqlalchemy — use directly when you need full ORM/Core power and don't want a thin abstraction hiding it; pair with hand-written Pydantic schemas.
- tortoise/tortoise-orm — use when you want an async-native ORM with a Django-like API and built-in Aerich migrations.
- pydantic/pydantic — the validation half alone; use with raw SQLAlchemy when you'd rather compose the two yourself than accept SQLModel's fusion.
- collerek/ormar — use when you want a Pydantic-based async ORM alternative that also targets FastAPI.
- encode/orm — use for a small async query-builder over SQLAlchemy Core when you don't need full ORM mapping.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2021-08 | Initial public release by tiangolo[^1]. |
| 0.0.x | 2021–2023 | Pydantic v1 + SQLAlchemy 1.4/2.0 era; slow release cadence. |
| 0.0.14 | 2023-12 | Pydantic v2 support added; Pydantic v1 dropped[^2]. |
| 0.0.16+ | 2024 | Repo moved under the `fastapi` GitHub organization. |
| 0.0.2x | 2025–2026 | Incremental fixes; still pre-1.0, still 0.0.x versioning. |

## References

[^1]: SQLModel documentation and project site. https://sqlmodel.tiangolo.com
[^2]: SQLModel release notes / changelog (Pydantic v2 support landed in the 0.0.14 line, late 2023). https://sqlmodel.tiangolo.com/release-notes/

## Tags

python, orm, sql, sqlalchemy, pydantic, fastapi, database, type-annotations, data-modeling, backend
