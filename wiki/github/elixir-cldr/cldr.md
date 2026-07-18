# elixir-cldr/cldr

> Elixir implementation of the Unicode CLDR — compile-time-generated locale data for formatting numbers, dates, units, and lists — now in managed sunset with support ending December 31, 2027.

[GitHub repo](https://github.com/elixir-cldr/cldr) ·
[Docs (HexDocs)](https://hexdocs.pm/ex_cldr) ·
[License: Apache-2.0](https://github.com/elixir-cldr/cldr/blob/main/LICENSE.md)

## Overview

`ex_cldr` (repo: `elixir-cldr/cldr`) brings the Unicode Consortium's Common Locale Data Repository[^1] to Elixir: locale-aware formatting and parsing of numbers, currencies, dates/times, units, lists, and language tags (BCP 47, `Accept-Language` headers). Started by Kip Cole in 2016, it grew into the de facto i18n data layer for the BEAM — a hub package plus roughly fifteen satellite packages (`ex_cldr_numbers`, `ex_cldr_dates_times`, `ex_cldr_units`, `ex_cldr_routes`, `ex_cldr_messages`, and so on), each compiled into your app via a provider mechanism[^2]. `ex_money` builds its currency handling on the same substrate.

The defining design choice is compile-time code generation. You declare a "backend" module with `use Cldr` and a list of locales; the library downloads versioned locale JSON at compile time and generates functions that encapsulate the data, trading longer compiles and larger BEAM files for fast runtime lookups. This is the opposite of gettext-style runtime catalogs, and it shapes everything: configuration, deployment, CI, and upgrade behavior.

The headline fact as of 2026: the project is in a planned wind-down. The README states support for `ex_cldr` and related libraries runs until December 31, 2027, and directs consumers to the `localize` package (Hex) as the replacement[^3]. Maintenance is still real — v2.47.5 shipped 2026-06-28 with zero open issues against 474 stars — but new adoption should weigh the sunset date against migration cost.

## Getting Started

```elixir
# mix.exs — on OTP 27+ / Elixir 1.18+ the JSON dep is unnecessary
defp deps do
  [
    {:ex_cldr, "~> 2.37"},
    {:jason, "~> 1.0"}
  ]
end
```

```elixir
defmodule MyApp.Cldr do
  use Cldr,
    locales: ["en", "fr", "ja"],
    default_locale: "en",
    providers: [Cldr.Number]   # requires {:ex_cldr_numbers, "~> 2.0"}
end

MyApp.Cldr.put_locale("fr")
MyApp.Cldr.Number.to_string!(1234.56, currency: :EUR)
# => "1 234,56 €"
```

The bare `ex_cldr` package only manages locale data and language tags; any actual formatting requires a satellite package wired in via `:providers`.

## Architecture / How It Works

**Backend modules.** `use Cldr` is a macro that generates a full API module at compile time from your locale list. Multiple backends can coexist (e.g., a library and the host app each with their own), which avoids the global-config collision problems of a singleton design. The cost: every locale you add is code that must be compiled, and the docs explicitly warn against `locales: :all` (~700 locales) as a multi-minute compile and a memory sink[^2].

**Providers.** Satellite packages expose `cldr_backend_provider/1`, which receives a `Cldr.Config` struct and returns an AST that is spliced into the backend module during compilation. This is why `MyApp.Cldr.Number` exists only if `Cldr.Number` is listed in `:providers` — the module is literally generated into your backend. It is an unusual, macro-heavy coupling model: flexible, but opaque to grep and to tools that expect modules to exist in source.

**Locale data at compile time.** Locale definitions are versioned JSON files downloaded during compilation into `priv/cldr` (configurable via `:data_dir`), keyed to the `ex_cldr` version. Recent minor versions track Unicode CLDR data releases: v2.38 shipped CLDR 45, v2.41 CLDR 47, v2.44 CLDR 48, v2.45 CLDR 48.1[^4].

**Configuration precedence** is global (`config :ex_cldr`) < otp_app config < backend module options, with only a handful of keys (`:json_library`, `:default_locale`, `:cacertfile`, …) valid globally. `:json_library` must be global because it is needed before any backend compiles; on OTP 27+ the built-in `:json` module is auto-detected and no configuration is needed[^5].

**Gettext interop.** A backend can point at a Gettext module as an additional locale source; `ex_cldr` transliterates POSIX-style names (`en_US`) into Unicode form (`en-US`). The two libraries are complementary: gettext translates message strings, `ex_cldr` formats data.

## Production Notes

**The sunset is the first thing to evaluate.** Support ends 2027-12-31 with `localize` as the designated successor[^3]. Existing apps have runway; greenfield projects are effectively choosing a migration up front.

**Compile-time locale downloads bite CI/CD.** Because locale files are fetched during compilation and are version-locked to `ex_cldr`, cached `_build`/`deps` directories in CI can pair stale locale data with a new library version. The `force_locale_download: true` option exists specifically for reproducible builds; air-gapped or proxied environments need `:https_proxy` / `HTTPS_PROXY` and a reachable download host at build time[^2].

**Compile time scales with locale count.** A handful of locales is cheap; wildcard configs like `"en-*"` (100+ regional variants) or `:all` are not. The common pattern is one locale in dev/test config and the full list only in prod config — supported via `:otp_app` configuration.

**Formatting output changes with data upgrades.** Bumping `ex_cldr` bumps the underlying CLDR data, and Unicode revises formats between releases. v2.44 (CLDR 48) also changed internal locale JSON structures — flagged as breaking even though most consumers never touch the raw data[^4]. Snapshot tests that assert exact formatted strings will churn on upgrades.

**Runtime-compiled formats are slow-path.** User-supplied format strings not listed in `:precompile_number_formats` (and the date/interval equivalents) compile at runtime at roughly half the speed of precompiled ones; the library can log warnings to surface candidates for precompilation[^2].

**Default locale is `en-001`**, not `en` — international English. Formatting differences from `en-US` (dates especially) surprise teams who never set `:default_locale`.

**JSON library history.** As of v2.37.2 the `:json_library` setting no longer falls back to reading Phoenix or Ecto configuration — apps that relied on that implicit inheritance broke quietly on upgrade[^5].

## When to Use / When Not

**Use when:**
- You maintain an existing Elixir app already on the `ex_cldr` family or `ex_money` — it remains supported through 2027 and actively patched.
- You need CLDR-grade data formatting (numbers, currencies, units, dates, plural rules) rather than just translated strings.
- You want per-app isolated i18n config (backend modules) instead of global state.

**Avoid when:**
- You are starting a new project — the maintainer directs new work to `localize`, and adopting a sunsetting library creates a known migration.
- You only need message translation: `gettext` alone is lighter and has no compile-time data download.
- Your build environment cannot reach the locale download host and you cannot vendor `priv/cldr` — compile-time fetching becomes an operational fight.
- Compile time is precious and you need many locales — the codegen model works against you.

## Alternatives

- localize (Hex package, no separate public GitHub repo located) — the designated successor from the same maintainer; use it for new Elixir projects per the official guidance[^3].
- elixir-gettext/gettext — use instead when your need is translated UI strings with plural forms, not locale-aware data formatting.
- unicode-org/icu4x — use when you want CLDR formatting from a Rust core (via NIF/FFI) shared across non-BEAM platforms.
- twitter/twitter-cldr-rb — the closest analogue in Ruby; relevant as a reference implementation, not a BEAM option.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2017-12-08 | First stable release[^6]. |
| 2.0.0 | 2018-11-22 | Backend-module rewrite (`use Cldr`); the current architecture[^6]. |
| 2.18.0 | 2020-11-01 | CLDR 38 data. |
| 2.24.0 | 2021-10-27 | CLDR 40 data. |
| 2.37.0 | 2023-04-28 | CLDR 43; 2.37.2 stops reading Phoenix/Ecto JSON config[^5]. |
| 2.38.0 | 2024-04-20 | CLDR 45 data. |
| 2.41.0 | 2025-03-15 | CLDR 47 data. |
| 2.44.0 | 2025-11-05 | CLDR 48; breaking changes to locale JSON structure[^4]. |
| 2.45.0 | 2026-01-17 | CLDR 48.1 data. |
| 2.47.5 | 2026-06-28 | Latest patch release; sunset notice active (support to 2027-12-31)[^3]. |

## References

[^1]: Unicode Consortium, Common Locale Data Repository. https://cldr.unicode.org
[^2]: `ex_cldr` README — configuration, providers, locale download, `:all` warning. https://github.com/elixir-cldr/cldr#readme
[^3]: elixir-cldr organization, "Future support" — support until 2027-12-31; migrate to `localize`. https://github.com/elixir-cldr#future-support · https://hex.pm/packages/localize
[^4]: `elixir-cldr/cldr` GitHub releases — CLDR data mapping and v2.44.0 breaking changes. https://github.com/elixir-cldr/cldr/releases
[^5]: `ex_cldr` README — `:json_library` configuration notes (v2.37.2 change; OTP 27 auto-detection). https://github.com/elixir-cldr/cldr#global-configuration
[^6]: `elixir-cldr/cldr` release tags v1.0.0 (2017-12-08), v2.0.0 (2018-11-22). https://github.com/elixir-cldr/cldr/releases

## Tags

elixir, i18n, l10n, cldr, unicode, localization, number-format, date-time-format, locale-data, library
