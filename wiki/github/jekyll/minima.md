# jekyll/minima

> Jekyll's default theme — the blog scaffold every `jekyll new` site starts
> from, and a long-running case study in released-gem vs. master version skew.

[GitHub repo](https://github.com/jekyll/minima) ·
[Theme preview](https://jekyll.github.io/minima/) ·
[License: MIT](https://github.com/jekyll/minima/blob/master/LICENSE.txt)

## Overview

Minima is a "one-size-fits-all" blog theme for Jekyll, extracted in May 2016
when Jekyll 3.2 introduced gem-based themes[^1]. It is what `jekyll new`
scaffolds, which makes it — by installed base, not by stars — one of the most
widely deployed pieces of front-end code in the static-site world. The GitHub
numbers reflect that scaffold role: ~3.8k stars and ~3.8k forks, an unusual
1:1 ratio produced by years of "fork the theme to customize your blog"
tutorials rather than by contributor interest.

The defining tension is version skew. The released gem lineage is 2.x
(v2.5.0 in 2018, v2.5.1 in 2019, and a maintenance v2.5.2 in 2024)[^2], while
`master` has spent years as an unreleased semver-major "Minima 3" rewrite —
skins, `base.html`, restructured Sass — that the README documents as if it
were current. The README itself opens with a warning that `master` is under
active development with non-backwards-compatible changes and should only be
consumed via a pinned git ref[^3]. Users routinely configure v3 features
against the v2.5.x gem and wonder why nothing happens.

Maintenance is real but slow: the repo saw pushes as recently as April 2026,
and issues are triaged, but release cadence is measured in years. Treat Minima
as stable-but-frozen rather than evolving.

## Getting Started

Add the gem to your Jekyll site's `Gemfile` and reference it in config:

```ruby
# Gemfile
gem "minima"
```

```yaml
# _config.yml
theme: minima
header_pages:        # v2.5.x; master/v3 uses minima.nav_pages instead
  - about.md
```

To use the unreleased v3 work, pin a specific commit — never track `HEAD`:

```yaml
# _config.yml (with jekyll-remote-theme plugin)
remote_theme: "jekyll/minima@1e8a445"
```

## Architecture / How It Works

Minima is a gem-based theme, which means its files live inside the installed
gem, not in your site tree. Jekyll's lookup order checks the site source
first, then the theme gem — so customization is copy-and-override: run
`bundle show minima` to find the gem path, copy the file you want (say
`_includes/head.html`) into your site, and your copy shadows the gem's[^4].
Nothing merges; you own the whole overridden file, including future upstream
fixes to it.

The structure is the standard `jekyll new-theme` scaffold:

- **Layouts** — `home.html` (post listing plus, from v2.2, any content from
  your `index.md` injected above it), `post.html`, `page.html`, all deriving
  from a base layout. The base is `default.html` in 2.x and renamed to
  `base.html` on master, a rename that breaks sites carrying a customized
  `_layouts/default.html` across the upgrade[^3].
- **Includes** — header/footer/head partials, Disqus comments, and Google
  Analytics; the latter two render only when `JEKYLL_ENV=production`.
- **Sass** — `_sass/minima/` partials with two designated override hooks:
  `custom-variables.scss` (variables only) and `custom-styles.scss` (rules
  only). Master adds a **skins** system — `classic`, `dark`, `auto`,
  `solarized` variants — where a skin file owns the color palette and syntax
  highlighting, and `auto` switches via `prefers-color-scheme`[^3].
- **Plugins** — minima 2.5.x declares `jekyll-feed` and `jekyll-seo-tag` as
  runtime dependencies and wires them into `head.html`.

There is no JavaScript. The theme's entire runtime surface is one compiled
stylesheet and Liquid-rendered HTML, which is why it survives Jekyll upgrades
that break heavier themes.

## Production Notes

- **The README documents a version you probably don't have.** Skins,
  `minima.nav_pages`, `base.html`, and the Font Awesome social icons are
  master-only. On the released 2.5.x gem you have `header_pages`,
  `default.html`, no skins, and a bundled SVG icon sprite. Read the README at
  your version's tag (e.g. `/blob/v2.5.0/README.md`), not on master[^3].
- **GitHub Pages pins the theme.** The classic Pages build environment ships
  minima 2.5.1 (with Jekyll 3.10)[^5]. You cannot get newer Minima there
  except through `jekyll-remote-theme` or by building with GitHub Actions and
  deploying the artifact.
- **Dark mode requires the unreleased version.** The `skin: dark` / `auto`
  settings do nothing on 2.5.x; on the released gem you hand-roll dark-mode
  CSS in an override file.
- **Sass deprecation noise.** Minima's SCSS predates dart-sass; under
  jekyll-sass-converter 3 the `@import`-based structure emits deprecation
  warnings. Harmless today, but the eventual `@import` removal in dart-sass
  will require upstream (or your fork) to migrate.
- **Environment-gated includes surprise people.** Disqus and Analytics render
  only with `JEKYLL_ENV=production`, so comments are invisible in local dev —
  frequently misreported as a bug.
- **Upgrade path is manual.** With copy-and-override customization there is no
  merge tooling: every overridden file silently pins its old markup. Audit
  your overrides against the gem's files on each theme bump.

## When to Use / When Not

**Use when:**

- You want a zero-configuration blog and intend to write, not theme.
- You are on GitHub Pages' classic build and want the supported default.
- You need a minimal, dependency-light base to override into a custom design.
- You are teaching Jekyll — it is the reference example of a gem-based theme.

**Avoid when:**

- You need dark mode, navigation config, or current docs on a released gem —
  the v2/v3 skew will cost you an afternoon.
- You want an actively released theme; expect years between gem versions.
- You need built-in search, TOCs, archives, or i18n — Minima has none.
- Your site is documentation rather than a blog.

## Alternatives

- mmistakes/minimal-mistakes — use instead when you want a batteries-included
  Jekyll theme (author profiles, search, taxonomies) at the cost of weight.
- cotes2020/jekyll-theme-chirpy — use instead for a modern blog with dark
  mode, search, and PWA support released today, not on an unreleased branch.
- just-the-docs/just-the-docs — use instead when the site is documentation
  with navigation and search rather than a post stream.
- pages-themes/cayman — use instead for a single-page project site on GitHub
  Pages' built-in theme roster.
- adityatelange/hugo-PaperMod — use instead if you are willing to leave
  Jekyll for Hugo's build speed with a similar minimal-blog aesthetic.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0 | 2016-05 | Extracted as Jekyll's default theme for the new gem-based theme system (Jekyll 3.2)[^1]. |
| v2.0.0 | 2016-10 | First major restructure of the 1.x scaffold[^2]. |
| v2.2.0 | 2018-01 | Home layout injects `index.md` content above the post list; post listing optional[^2]. |
| v2.5.0 | 2018-04 | Last feature release of the 2.x line[^2]. |
| v2.5.1 | 2019-08 | Maintenance release; still the version pinned by GitHub Pages[^5]. |
| v2.5.2 | 2024-09 | Maintenance release after a five-year gap[^2]. |
| v3 (master) | unreleased | Skins, `base.html`, `minima.nav_pages`, Font Awesome social icons; consume only via pinned ref[^3]. |

## References

[^1]: Jekyll 3.2.0 release announcement introducing gem-based themes — 2016-07-26. https://jekyllrb.com/news/2016/07/26/jekyll-3-2-0-released/
[^2]: jekyll/minima releases (dates per GitHub). https://github.com/jekyll/minima/releases
[^3]: minima README, master-branch warning and Minima 3 migration notes. https://github.com/jekyll/minima#readme
[^4]: Jekyll docs, "Themes — Overriding theme defaults". https://jekyllrb.com/docs/themes/
[^5]: GitHub Pages dependency versions. https://pages.github.com/versions/

## Tags

scss, jekyll, jekyll-theme, static-site, blogging, github-pages, ruby-gem, liquid, css-theme, default-theme
