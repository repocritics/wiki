# sharkdp/pastel

> A command-line tool to generate, analyze, convert, and manipulate colors across RGB, HSL, CIELAB, and CIELCh.

[GitHub repo](https://github.com/sharkdp/pastel) ·
[License: Apache-2.0](https://github.com/sharkdp/pastel/blob/master/LICENSE-APACHE) (dual Apache-2.0 / MIT)

## Overview

`pastel` is a Unix-philosophy color utility by David Peter (sharkdp), the same
author behind `bat`, `fd`, and `hyperfine`[^1]. It treats a color as a value
that can be piped, transformed, and composed the way you would compose text with
`grep` and `sed`. Each subcommand — `saturate`, `mix`, `lighten`, `distinct`,
`paint` — reads colors from arguments or stdin and writes colors to stdout, so
chains like `pastel random | pastel mix red | pastel format hex` are the intended
usage pattern, not a novelty.

The defining design choice is that operations are defined in perceptually
uniform color spaces rather than in raw sRGB. Lightening, mixing, and distance
calculations happen in CIELAB / CIELCh, which is why `pastel lighten 0.2` and
`pastel distinct 8` produce results that look right to the eye instead of merely
being arithmetically even in RGB. The tradeoff is that pastel is a color-math
and terminal tool, not an image tool: it manipulates individual colors and color
sets, not pixels, files, or palettes embedded in design documents.

The project is mature and effectively feature-complete. Commits still land
(the default branch saw activity into 2026[^2]), but this is low-churn
maintenance — dependency bumps, packaging, and occasional fixes — rather than
active feature development. Treat it as stable infrastructure, not a
fast-moving project.

## Getting Started

```bash
# macOS
brew install pastel
# Arch
sudo pacman -S pastel
# From source (needs Rust 1.83+)
cargo install pastel
```

```bash
# Convert sRGB hex to HSL
pastel format hsl ff8000

# Show and analyze a color block in the terminal
pastel color "rgb(255,50,127)"

# Generate 8 maximally distinct colors and print them as hex
pastel distinct 8 | pastel format hex

# Compose: pick a readable foreground for a given background
bg="hotpink"
fg="$(pastel textcolor "$bg")"
pastel paint "$fg" --on "$bg" "well readable text"
```

Colors can be given as CSS names (`lightslategray`), hex (`#778899`, `778899`,
`789`), `rgb(...)`, `hsl(...)`, or bare comma triples. A `-` argument reads that
position from stdin, which is how commands are chained.

## Architecture / How It Works

pastel is split into a reusable `pastel` library crate (the color model and
conversions) and the CLI that wraps it. Internally a color is carried in a
canonical form and converted on demand between sRGB, HSL, CIELAB, and CIELCh;
the perceptual spaces are the ones used whenever an operation needs to reason
about how a color *looks* rather than how it is encoded.

Notable internals:

- **Perceptual operations.** `lighten` / `darken` adjust the L channel in a
  Lab-like space, and `mix` interpolates in a chosen space (Lab by default),
  so a 50% mix of two colors lands near the perceptual midpoint rather than the
  RGB midpoint.
- **`distinct`.** Generating N visually distinct colors is an optimization
  problem: pastel iteratively maximizes the minimum pairwise color difference
  (CIE color-difference metric) over the set, which is why the command takes a
  moment and why results vary run to run.
- **`paint` / `color`.** Output is ANSI escape sequences. pastel downsamples to
  the terminal's capability — 24-bit truecolor where available, otherwise the
  ANSI 8-bit (256-color) palette — and honors conventions for disabling color.
- **`pick`.** Picking a color off the screen is delegated to an external
  screen color picker; on Linux this depends on a helper such as `gpick` or
  `xcolor` being installed, so `pick` is the one subcommand with an environment
  dependency beyond the binary itself.

Because everything is a color-in / color-out subcommand, the CLI surface is
essentially a thin dispatch layer over the library, and the compositional power
comes from stdin/stdout plumbing rather than from any internal pipeline engine.

## Production Notes

- **It's a workflow/scripting tool, not a service.** pastel shines inside shell
  scripts, CI banners, and dotfiles (`pastel paint` for colored log prefixes).
  There is no daemon, config file, or plugin system to operate.
- **`pick` is not portable.** On headless machines, containers, or CI it will
  not work — there is no screen to sample and no picker helper installed. Guard
  any script that calls it.
- **Truecolor detection.** In terminals or multiplexers that misreport color
  capability (some `tmux`/`screen` setups, certain SSH sessions), `paint` output
  can degrade to 256-color or look wrong. Verify `$COLORTERM` and terminal
  settings before assuming 24-bit output in automation.
- **`distinct` is nondeterministic and unordered.** Do not hard-code an
  expectation that a given N always yields the same colors; pin colors explicitly
  if you need reproducibility.
- **Build requirement.** Installing from source needs a reasonably recent Rust
  toolchain (1.83+ as of the current release); distro packages avoid this. The
  crate name on crates.io is `pastel`, but note an unrelated `pastel` GUI project
  exists elsewhere — the binary here is the sharkdp CLI.
- **Color science caveats.** pastel uses sRGB assumptions and standard CIE
  formulae; it is not a color-managed pipeline and does not handle ICC profiles,
  wide-gamut displays, or non-sRGB working spaces. For print or
  color-managed design work, reach for a dedicated tool.

## When to Use / When Not

**Use when:**
- You want to convert, mix, lighten, saturate, or name colors from the shell.
- You need programmatically generated, visually distinct palettes for charts,
  tags, or terminal UIs.
- You're colorizing shell/CI output and want a readable-foreground helper.
- You want a single static binary with no runtime dependencies for the core commands.

**Avoid when:**
- You need image or pixel manipulation (use ImageMagick).
- You need a color-managed / ICC-aware pipeline for print or wide-gamut work.
- You want an interactive GUI palette editor rather than a scriptable CLI.
- You're on a headless box and depend on `pastel pick`.

## Alternatives

- ImageMagick — full raster image manipulation, including per-pixel color ops, when you need to touch actual images rather than color values.
- chroma.js — use instead when you need color math as a JavaScript library inside an application rather than a CLI.
- python-colormath / colour-science — use when your color work lives in a Python data/science pipeline and needs programmatic control.
- gpick — use for interactive, GUI-based on-screen color picking and palette building (and as pastel's `pick` backend).
- colorls / lscolors — unrelated to color math; named here only to disambiguate, they colorize directory listings, not colors themselves.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-06 | First public release by sharkdp; CLI + `pastel` library crate.[^3] |
| 0.12.0 | current | Latest tagged release; requires Rust 1.83+ to build from source.[^4] |

(Intermediate 0.x releases exist between these points; exact dates omitted where
not verified.)

## References

[^1]: David Peter (sharkdp) GitHub profile — author of bat, fd, hyperfine, pastel. https://github.com/sharkdp
[^2]: Repository metadata (default branch `master`, last push 2026-05-01), GitHub API, retrieved 2026-07. https://github.com/sharkdp/pastel
[^3]: Repository creation date 2019-06-02, GitHub API. https://github.com/sharkdp/pastel
[^4]: pastel README, installation section (v0.12.0 Debian package, Rust 1.83 build requirement). https://github.com/sharkdp/pastel#installation

## Tags

rust, cli, color, color-space, cielab, terminal, command-line, ansi-colors, palette-generation, developer-tools
