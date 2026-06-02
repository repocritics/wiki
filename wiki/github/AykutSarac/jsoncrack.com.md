# AykutSarac/jsoncrack.com

A web app that visualizes JSON, YAML, XML, and CSV data as interactive graphs — purpose-built for understanding nested structures.

## What it is

A TypeScript / React + Next.js application that takes structured-data input and renders it as a navigable node-graph. Useful for inspecting deeply-nested API responses, configuration files, or unfamiliar data exports. Hosted at jsoncrack.com; OSS code is the canonical reference plus a self-hostable instance.

## Key features

- Multi-format input: JSON, YAML, XML, CSV — auto-detected.
- Pan + zoom + node-collapse on the rendered graph.
- Search, type detection, schema inference.
- Export to image / PDF / SVG.
- Apache 2.0 licensed.

## Tech stack

- TypeScript primary.
- React + Next.js.
- React Flow / custom canvas rendering for the graph.

## When to reach for it

- You're debugging a complex JSON response and want a visual map.
- You're documenting API shapes and want a quick visualization.
- You're teaching JSON / YAML basics and want a learning aid.

## When *not* to reach for it

- You're working with very large JSON (10MB+) — rendering becomes sluggish.
- You want diff-style comparison of two JSONs — that's a different tool (JSON Diff).

## Maturity signal

44k stars, 3.5k forks, Apache 2.0, actively maintained.

## Alternatives

- JSONCrack-like alternatives (`treedoc`, `jsonhero`).
- Browser DevTools JSON viewer — built-in but less navigable.
- jq + custom CLI for command-line exploration.

## Tags

typescript, react, nextjs, data-visualization, json, yaml, csv, developer-tools, apache-license
