# spatie/laravel-permission

> Database-backed roles and permissions for Laravel, layered on top of the framework's own authorization gate.

[GitHub repo](https://github.com/spatie/laravel-permission) ·
[Official docs](https://spatie.be/docs/laravel-permission) ·
[License: MIT](https://github.com/spatie/laravel-permission/blob/main/LICENSE.md)

## Overview

`laravel-permission` is Spatie's package for associating users with roles and permissions stored in the database. It is one of the most-installed packages in the Laravel ecosystem (tens of millions of Packagist downloads) and, for most teams, the default answer to "how do I do RBAC in Laravel." The repo dates to 2015 and traces its lineage to a Laracasts lesson by Jeffrey Way[^1].

The design decision that defines the package: it does not invent its own authorization API. Permissions are registered onto Laravel's native authorization gate, so you check them with the framework's own `$user->can()`, `@can` Blade directive, and policy machinery rather than package-specific calls[^2]. The package supplies the storage layer (roles, permissions, and the pivot tables joining them to models) plus convenience traits and middleware; the enforcement layer stays Laravel's. This keeps it composable with policies and gates but also means the two systems can disagree in confusing ways (see Production Notes).

The recurring tension is that RBAC looks trivial in the README and gets subtle fast in production: guard names, permission caching, multi-tenant "teams," and the distinction between a role and a permission all become footguns once an app has real users. The package is honest infrastructure, not a policy framework — it answers "does this user have permission X" and leaves "what should X mean" to you.

## Getting Started

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

Add the `HasRoles` trait to your `User` model, then create and assign roles/permissions:

```php
use Spatie\Permission\Traits\HasRoles;
use Spatie\Permission\Models\Role;
use Spatie\Permission\Models\Permission;

class User extends Authenticatable
{
    use HasRoles;
}

$role = Role::create(['name' => 'writer']);
$permission = Permission::create(['name' => 'edit articles']);

$role->givePermissionTo($permission);   // role -> permission
$user->assignRole('writer');            // user -> role

// Because permissions register on Laravel's gate:
$user->can('edit articles');            // true
```

Roles and permissions are plain Eloquent models, so seeders are the normal way to define an app's permission set.

## Architecture / How It Works

The storage model is three pivot tables around two entity tables. `roles` and `permissions` hold the definitions; `role_has_permissions` joins them; and `model_has_roles` / `model_has_permissions` are **polymorphic** pivots that attach roles/permissions to any model (usually `User`, but the morph columns mean it is not user-specific). Because assignment is polymorphic, you can grant permissions directly to a user, bypassing roles entirely — direct permissions and role-derived permissions are unioned at check time.

The `HasRoles` trait composes `HasPermissions` and mixes in the query scopes, relationship accessors, and assignment methods. On boot, the package's `PermissionRegistrar` loads all permissions and registers them onto Laravel's `Gate`, which is what makes `$user->can()` resolve package permissions[^2].

**Caching is central to the design.** Loading every permission on every request would be a query storm, so the registrar caches the full permission set (default TTL 24 hours, configurable, keyed under `spatie.permission.cache`). The cache is automatically flushed when you create or modify permissions/roles through the package's models. It is *not* flushed if you mutate the underlying tables directly (raw SQL, a different model), which is the single most common source of "my permission change didn't take effect" reports[^3].

**Guards** are a first-class concept. Every role and permission has a `guard_name` (default `web`). Checks only match within the same guard, so an `api`-guard user will not see a `web`-guard role even with an identical name. This mirrors Laravel's multi-guard auth but surprises people running an API and a web session side by side.

**Teams** (multi-tenancy) are opt-in via `config('permission.teams')`. Enabling it adds a `team_id` scope so the same user can hold different roles in different teams. It changes migration shape and cache semantics, so it must be decided before the first migrate rather than bolted on later[^4]. **Wildcard permissions** are an optional matching mode where `articles.*` satisfies `articles.edit`, trading explicitness for flexibility. The package also supports UUID/ULID primary keys and enum-based permission names on modern Laravel/PHP.

## Production Notes

- **The cache is the top operational footgun.** Permission changes made outside the package's models (direct DB writes, imports, another service) will not invalidate the cache; call `php artisan permission:cache-reset` or `forgetCachedPermissions()` afterward[^3]. In tests, seeding permissions and then asserting in the same request often needs a manual cache reset because the registrar cached the empty set at boot.
- **Guard mismatches fail silently.** Assigning a `web`-guard role and checking under the `api` guard returns `false` with no error. When roles "don't work," the guard name is the first thing to verify.
- **N+1 on role/permission access.** Iterating users and calling `$user->hasRole()` / `getAllPermissions()` without eager loading hammers the DB. Eager-load the `roles` and `permissions` relations; the cache helps for permission *definitions* but not for the per-user pivot joins.
- **Super-admin should bypass, not be granted every permission.** The documented pattern is a `Gate::before` returning `true` for an admin role rather than attaching thousands of permissions. Note this also short-circuits policy denials, which can mask bugs.
- **Blade `@can` vs `@role`.** `@can` goes through the gate (respects `Gate::before`, policies, super-admin bypass); `@role`/`@hasrole` check role membership directly and ignore the gate. Mixing them inconsistently produces UI that disagrees with server-side authorization.
- **Middleware registration changed across Laravel versions.** The `role`, `permission`, and `role_or_permission` route middleware need aliasing; Laravel 11's slimmer `bootstrap/app.php` moved where that happens, so upgrade guides differ by framework version[^4].
- **Enabling teams later is a migration.** Turning on `teams` after data exists means backfilling `team_id`; plan it up front for multi-tenant apps.

## When to Use / When Not

**Use when:**
- You need database-managed roles/permissions in a Laravel app and want to keep using Laravel's native `can`/policies for enforcement.
- Admins should edit roles and permissions at runtime, not hard-coded in policy files.
- You want multi-guard or multi-tenant (teams) RBAC without hand-building the pivot layer.

**Avoid when:**
- Your authorization is genuinely attribute/ownership-based ("user can edit *their own* post") — that is Laravel policies, and this package adds little.
- You only have two or three static roles; a config array plus `Gate::define` may be simpler than three extra tables and a cache to reason about.
- You need fine-grained per-record ABAC or relationship-based access — reach for a policy engine or purpose-built authz service instead.

## Alternatives

- JosephSilber/bouncer — fluent RBAC with per-model "abilities" and built-in scoping; use when you want ownership-scoped abilities out of the box.
- santigarcor/laratrust — similar roles/permissions with first-class team support; use when multi-tenancy is the primary requirement.
- zizaco/entrust — older, wildcard-pattern permissions; largely legacy, use only when maintaining an existing Entrust app.
- ultraware/roles — archived; a different structural take, not for new projects.
- Laravel policies + gates (framework built-in) — use when authorization is logic/ownership-based rather than a stored role/permission catalog.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2016 | Initial release; based on a Laracasts roles/permissions lesson[^1]. |
| 2.x | 2017 | Major rewrite; multi-guard support (Alex Vanderbist contributed)[^5]. |
| 3.x | ~2019 | Tracking newer Laravel; permission caching improvements. |
| 4.x | ~2020 | Laravel 8 support; wildcard permission option. |
| 5.x | ~2021 | Teams / multi-tenancy support; UUID model support. |
| 6.x | ~2023 | Laravel 10+, PHP 8.1+, enum-based permission names[^4]. |

## References

[^1]: README credits — package based on Jeffrey Way's Laracasts lessons on roles and permissions. https://github.com/spatie/laravel-permission#credits
[^2]: Spatie docs, "Basic Usage" — permissions register on Laravel's Gate; check with `can()`. https://spatie.be/docs/laravel-permission/basic-usage/basic-usage
[^3]: Spatie docs, "Cache" — permission cache and manual reset. https://spatie.be/docs/laravel-permission/advanced-usage/cache
[^4]: Spatie docs, "Installation" and "Teams/Multitenancy". https://spatie.be/docs/laravel-permission
[^5]: README credits — Alex Vanderbist thanked for v2 help. https://github.com/spatie/laravel-permission#credits

## Tags

php, laravel, authorization, rbac, roles-permissions, access-control, eloquent, security, middleware, multi-tenancy
