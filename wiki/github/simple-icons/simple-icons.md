# simple-icons/simple-icons

> Monochrome single-path SVG icons for popular brands — one canonical logo per brand, dedicated to the public domain.

[GitHub repo](https://github.com/simple-icons/simple-icons) ·
[Official website](https://simpleicons.org) ·
[License: CC0-1.0](https://github.com/simple-icons/simple-icons/blob/develop/LICENSE.md)

## Overview

Simple Icons is a curated collection of over 3,400 brand and product logos, each expressed as a single-color, single-`<path>` SVG on a 24×24 viewBox[^1]. It is not a general-purpose UI icon set (arrows, gears, chevrons) — it is specifically the logos of companies, products, protocols, and open-source projects, in the monochrome silhouette form used for footers, "built with" badges, social links, and tech stacks.

The project's defining constraint is deliberate uniformity: every icon is reduced to one color and one path so it renders consistently at any size and inherits its color from the surrounding page (`fill: currentColor`). This makes it trivial to theme, but it excludes any logo that cannot survive being flattened to a single silhouette — multicolor marks, gradients, and wordmark-plus-symbol lockups are either simplified or rejected.

The second defining tension is legal, not technical. The repository's code and SVG data are released under CC0-1.0 (public-domain dedication), but the *logos themselves remain the trademarks of their owners*. CC0 waives the project's copyright in the drawing; it does not and cannot grant you trademark rights. The maintainers ask every user to read a separate legal disclaimer before shipping icons, because displaying a brand mark can still imply endorsement or infringe trademark depending on context[^2]. This is the single most misunderstood thing about the project.

## Getting Started

Via npm (tree-shakeable — import only the slugs you use):

```bash
npm install simple-icons
```

```javascript
import { siGithub } from 'simple-icons';

console.log(siGithub.hex);   // "181717" — brand color, no leading '#'
console.log(siGithub.title); // "GitHub"
console.log(siGithub.svg);   // full <svg> string
console.log(siGithub.path);  // the raw path 'd' attribute
```

Via CDN (no build step; `[SLUG]` is the icon slug, optional color is a hex value):

```html
<img height="32" src="https://cdn.simpleicons.org/github" />
<img height="32" src="https://cdn.simpleicons.org/github/orange" />
<img height="32" src="https://cdn.jsdelivr.net/npm/simple-icons@v16/icons/github.svg" />
```

## Architecture / How It Works

The repository is a data set with a thin build layer, not a runtime library.

- **Two sources of truth per icon.** The drawing lives as a `.svg` file under `icons/`; the metadata (title, brand hex color, `source` URL, optional `aliases`, `guidelines`, `license`) lives in `data/simple-icons.json`. Neither is generated from the other — both are hand-maintained and cross-checked in CI.
- **SVG constraints.** Each file must contain exactly one `<path>`, a `viewBox` of `0 0 24 24`, `role="img"`, no `fill` attributes (color is left to the consumer), and be optimized with SVGO. The verify workflow rejects files that add a second path, embed color, or fail the optimizer's idempotency check.
- **Slugs are computed, not free-form.** A deterministic algorithm derives the slug from the title: lowercase, transliterate accented characters, strip most punctuation, and disambiguate collisions with documented rules (e.g. `.` → `dot`, `+` → `plus`). This is why the icon for ".NET" is `dotnet` and "Node.js" is `nodedotjs`. The slug is the public API surface; getting it wrong is a breaking change.
- **Build outputs.** Scripts assemble the npm package (`index.mjs`/`index.js` with `si`-prefixed exports plus TypeScript definitions), the Packagist package, and the JSON consumed by the website and the `cdn.simpleicons.org` color service.
- **Default branch is `develop`.** Releases are cut from it; badges and raw-content URLs commonly reference `develop` rather than a `main`/`master` branch.

Inclusion is gated by a notability policy: an icon is accepted only if the brand meets a popularity threshold and an authoritative vector source for the logo exists[^3]. Requests below the bar are declined, which keeps the set curated but makes it a poor fit if you need a long-tail or internal logo.

## Production Notes

**Pin the major version, or expect 404s.** Icons are *removed* as well as added — when a brand rebrands, disappears, or drops below the notability threshold, its icon is deleted in the next major release. Consumers on `@latest` or an unpinned CDN URL will get a broken image the day that happens. Pin to a major (`simple-icons@v16`, `cdn.jsdelivr.net/npm/simple-icons@v16/...`) and upgrade deliberately. Major versions ship frequently — roughly monthly — precisely because removals are breaking and are batched into majors under semver[^4].

**Tree-shaking is not optional.** The full package materializes 3,400+ icon objects, each carrying its complete SVG string. A naive `import * as icons from 'simple-icons'` pulls the entire set into your bundle. Import named slugs (`import { siGithub } from 'simple-icons'`) and rely on a bundler that eliminates dead exports; otherwise expect a multi-megabyte payload.

**Colors are single-hex and semantic, not decorative.** The `hex` field is the brand's primary color with no `#`. There is exactly one per icon — you cannot get the multicolor original mark from this project. If your design needs the full-color logo (e.g. the multi-hue Google "G"), Simple Icons is the wrong source.

**Slug drift.** Slugs are stable in intent but can change when a brand renames; the JSON data file, not memory, is the authority. Generate your icon references from the shipped data rather than hardcoding slugs across a large codebase.

**Trademark, again.** Because usage is trademark-governed, avoid patterns that imply a partnership you do not have (placing a brand's icon on your pricing page, in an app-store listing, or beside "official"). Some icons carry a `license` or `guidelines` field pointing at the brand's own usage rules — respect them. The CC0 license on the repo does not shield you here.

**No accessibility text by default.** The SVGs set `role="img"` but no `<title>`/`aria-label`. Add your own accessible name per usage; a bare icon is invisible to screen readers.

## When to Use / When Not

**Use when:**
- You need consistent monochrome brand logos for social links, footers, badges, or a "tech stack" strip.
- You want icons that inherit page color and theme cleanly in light/dark mode.
- You want a permissive, dependency-light source with npm, Packagist, and CDN distribution.

**Avoid when:**
- You need general UI/interface icons (menus, arrows, toggles) — this set has none.
- You need full-color original brand marks or wordmarks.
- You need a niche, internal, or non-notable logo that will not clear the inclusion policy.
- You are shipping brand logos in a context where trademark/endorsement risk matters and you have not cleared usage.

## Alternatives

- lucide/lucide — general-purpose interface icons (not brand logos); use when you need UI glyphs rather than company logos.
- FortAwesome/Font-Awesome — bundles brand icons (`fa-brands`) alongside a large UI set; use when you want one all-in-one library and accept the free/pro split.
- gilbarbara/logos — full-color, original SVG brand logos; use when you need the real multicolor mark instead of a monochrome silhouette.
- edent/SuperTinyIcons — extremely small multicolor brand SVGs; use when byte-for-byte payload size is the priority.
- devicons/devicon — programming languages, frameworks, and dev tools with plain/colored variants; use when you specifically want developer-tooling logos.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2012-11 | Repository created[^1]. |
| — | 2017–2019 | npm and Packagist distribution; single-path/24×24 SVG standard settled. |
| — | ~2021 | `cdn.simpleicons.org` color/dark-mode CDN service. |
| v16 | 2026 | Current major series; 3,400+ icons, default branch `develop`. |

Major versions are released frequently (about monthly) because icon removals are breaking and must land under semver; exact per-version dates are on the GitHub Releases page.

## References

[^1]: Simple Icons — repository and website. https://simpleicons.org
[^2]: Simple Icons legal disclaimer (trademarks remain with their owners despite CC0 on the data). https://github.com/simple-icons/simple-icons/blob/develop/DISCLAIMER.md
[^3]: Contributing guide — icon request and notability criteria. https://github.com/simple-icons/simple-icons/blob/develop/CONTRIBUTING.md
[^4]: Simple Icons releases (major-version cadence, additions and removals). https://github.com/simple-icons/simple-icons/releases

## Tags

svg, icons, brand-icons, logos, icon-pack, design-assets, javascript, npm, cc0, cdn
