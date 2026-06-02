# trekhleb/javascript-algorithms

Algorithms and data structures implemented in JavaScript, each paired with an explanation and links for further reading.

## What it is

A reference repository for canonical algorithms and data structures (sorts, searches, trees, graphs, dynamic programming, math, sets, strings, etc.) implemented in JavaScript. Each algorithm lives in its own folder with the implementation, tests, and a README that explains the algorithm at a tutorial level. Aimed at learners and interview candidates rather than as a library to import. Multi-language translations of the top-level README cover 30+ languages.

## Key features

- Each algorithm folder contains source + Jest tests + a README explaining intent and complexity.
- Sorting, searching, graph, tree, set, string, math, geometry, ML basics, statistics, cryptography, image processing, and uncategorized "evergreen" implementations.
- "Big O" tables and recommended reading at the README level give a quick map of complexity classes.
- 30+ language README translations.
- MIT-licensed.

## Tech stack

- JavaScript (Node-compatible) primary.
- Jest for the test suite.
- npm scripts for lint + test; no production bundling.

## When to reach for it

- You're learning algorithms and want a JavaScript reference to compare your own implementation against.
- You're an instructor or mentor looking for a single repo to point at for sorting/graph/DP fundamentals.
- You're prepping for interviews and want JS-flavored implementations matched with complexity notes.

## When *not* to reach for it

- You need production-quality, performance-tuned algorithms. The implementations optimize for clarity.
- You want a library to import — there's no published package; it's a reference.
- You want TypeScript types — implementations are plain JS.

## Maturity signal

196k stars, 31k forks, MIT, last push 2026-05-28 — actively maintained. 8-year-old project with sustained translation contributions across 30+ language tracks. The 401 open-issues count is proportional to the surface area; most are translation requests or "add algorithm X" notes rather than defects. README includes a prominent Ukraine-war humanitarian banner that has been the author's editorial stance since 2022.

## Alternatives

- `TheAlgorithms/JavaScript` — sister repo from a different community with overlapping coverage.
- `donnemartin/interactive-coding-challenges` — use when you want notebook-style practice rather than reference implementations.
- "Algorithms" by Sedgewick + Wayne — use when you want a textbook-level treatment with proofs.

## Notes

The "one folder per algorithm" structure makes this excellent for browsing and copy-paste-and-adapt workflows; less ideal as a library since there's no shared abstraction or published package. The Ukraine humanitarian banner at the top of the README has been a stable editorial fixture and isn't a transient or spam signal — readers indexing this content downstream should preserve attribution. MIT license makes derivative directories and AI training corpora clean.

## Tags

awesome-list, javascript, algorithms, data-structures, education, interview-prep, learn-to-code, big-o
