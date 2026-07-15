# chaijs/chai

> A BDD/TDD assertion library for Node and the browser that pairs with any test runner.

[GitHub repo](https://github.com/chaijs/chai) ·
[Official website](https://www.chaijs.com) ·
[License: MIT](https://github.com/chaijs/chai/blob/main/LICENSE)

## Overview

Chai is an assertion library, not a test runner. It provides the vocabulary you
use inside test cases — `expect(x).to.equal(y)`, `assert.deepEqual(a, b)` — and
leaves running the tests, reporting, and watch mode to something else (Mocha,
Karma, or a plain script). It has been around since 2011[^1], originally written
by Jake Luer, and is now maintained by Keith Cirkel and a small core team[^2].

Its defining characteristic is offering three assertion styles over one engine:
`assert` (a classical, Node-`assert`-like functional API), `expect` (chainable
BDD), and `should` (chainable BDD that mutates `Object.prototype` so every value
gains a `.should` getter). All three resolve to the same underlying `Assertion`
objects, so a plugin written once extends all styles. This flexibility is why
Chai became the default pairing for Mocha and why its assertion core was adopted
wholesale by Vitest, whose `expect` is Chai plus a Jest-compatibility layer[^3].

The central tension in 2026 is distribution, not features. Chai 5 (January 2024)
went ESM-only and dropped the bundled UMD `chai.js`, requiring Node 18+[^4]. That
was the right long-term call but stranded the large population of CommonJS test
suites that still `require('chai')`. Much of the ecosystem's Chai friction today
is a packaging problem, not an assertion problem.

## Getting Started

```bash
npm install --save-dev chai
```

```js
import { expect } from 'chai';

describe('array', () => {
  it('contains the expected members', () => {
    const beatles = ['john', 'paul', 'george', 'ringo'];
    expect(beatles).to.be.an('array').with.lengthOf(4);
    expect(beatles).to.include('paul');
    expect(beatles).to.not.include('pete');
  });
});
```

Run it with a runner, e.g. Mocha: `mocha spec.mjs`. The `assert` and `should`
styles cover the same ground:

```js
import { assert } from 'chai';
assert.lengthOf(['a', 'b'], 2);

import { should } from 'chai';
should();                 // adds `.should` to Object.prototype
[1, 2, 3].should.have.lengthOf(3);
```

## Architecture / How It Works

Every BDD assertion begins by wrapping a target in an `Assertion` object. Chained
words fall into two categories. **Language chains** — `to`, `be`, `been`, `is`,
`that`, `which`, `and`, `has`, `have`, `with`, `at`, `of`, `same`, `but`, `does` —
are no-op getters that exist purely for readability. **Assertion words** —
`equal`, `include`, `a`/`an`, `throw`, `true`, `lengthOf` — actually run a check
and throw an `AssertionError` on failure. Some assertion words are methods
(`equal(x)`); others are terminal getters (`.true`, `.ok`, `.exist`) that assert
merely by being accessed.

Flags carry state across the chain. `.not` sets a negate flag; `.deep` swaps
equality for structural comparison; `.nested`, `.own`, `.ordered` similarly
modify the next assertion. This is why `expect(x).to.not.equal(y)` works: `not`
mutates a flag that `equal` reads.

Deep equality is delegated to `deep-eql`, error inspection to `check-error`,
value rendering in messages to `loupe`, and property-path access (`.nested`) to
`pathval` — all separate chaijs packages[^5]. Chai's own code is mostly the chain
machinery and the assertion definitions.

Plugins extend the same core through `use(plugin)` and the builder API:
`Assertion.addMethod`, `addProperty`, `overwriteMethod`, and `addChainableMethod`.
Because all three styles share the `Assertion` prototype, `chai.use(sinonChai)`
makes `expect(spy).to.have.been.calledOnce` available everywhere at once. This
single-core, many-styles design is Chai's most durable idea.

## Production Notes

**The ESM cutover is the dominant operational fact.** Chai 5+ ships no CommonJS
entry point. A CommonJS suite that does `const chai = require('chai')` will fail
under Chai 5; the practical options are (a) stay on Chai 4.x, which remains
functional but no longer receives features, (b) convert the suite to ESM, or (c)
use dynamic `import()` inside an async setup. Jest projects on CommonJS are the
most affected, since Jest's ESM support is still flagged; many teams simply pin
`chai@4`[^4].

**`should()` mutates `Object.prototype`.** It works by adding a `.should` getter
to every object, which is elegant until you assert on `null`/`undefined` (no
prototype to attach to — use `should.exist(x)` instead), iterate object keys, or
share a realm with code sensitive to prototype pollution. `expect` avoids all of
this and is the recommended default.

**Getter assertions are an easy footgun.** `.true`, `.ok`, `.exist`, and friends
assert on property *access*, so `expect(x).to.be.true;` (no parentheses) is
correct — and `expect(x).to.be.true()` throws a TypeError because the boolean
isn't callable. Conversely, forgetting the terminal word entirely, e.g.
`expect(x).to.be;`, silently asserts nothing. Chai 4.2 added a lint-style guard
(`config.showDiff`, plus the `dirty-chai` community plugin) precisely because
these no-op assertions pass CI unnoticed[^6].

**Negation can over-pass.** `expect(1).to.not.equal(2).and.not.equal(3)` is fine,
but combining `.not` with type checks can assert less than you intend — a
negated assertion is satisfied by many values. Prefer positive assertions where
possible.

**If you use Vitest, you already have Chai.** Vitest's `expect` is Chai under the
hood with Jest matchers layered on; importing `chai` separately is usually
redundant and can produce two assertion-error types in one suite[^3].

## When to Use / When Not

**Use when:**
- You run Mocha (or a bare script/Karma) and need assertions to go with it.
- You want to choose between `assert`, `expect`, and `should` styles per team taste.
- You rely on the plugin ecosystem (sinon-chai, chai-as-promised, chai-http).
- Your project is ESM or you can pin `chai@4` for CommonJS.

**Avoid when:**
- You're on Jest or Vitest — both ship a built-in `expect`; adding Chai duplicates it.
- You need a batteries-included runner + assertions + mocks in one tool.
- You're locked to CommonJS and cannot pin to Chai 4 long-term.
- You want zero dependencies — Node's built-in `node:assert` may suffice.

## Alternatives

- jestjs/jest — use instead when you want runner, assertions (`expect`), and mocking in one tool with snapshot testing.
- vitest-dev/vitest — use instead when you want a Vite-native runner whose Chai-based `expect` is already bundled.
- power-assert-js/power-assert — use instead when you prefer descriptive failure diagrams over an assertion vocabulary.
- unexpectedjs/unexpected — use instead when you want a single extensible `expect` with a plugin-first design and rich diffs.
- Node's built-in `node:assert` — use instead when you want zero dependencies and don't need chainable BDD syntax.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013-06 | First stable release; assert/expect/should styles established. |
| 3.0 | 2015-10 | Assertion internals refactor; stricter equality behavior. |
| 4.0 | 2017-05 | Major release; `deep-eql` rewrite, improved deep/nested flags[^7]. |
| 5.0 | 2024-01 | ESM-only, UMD bundle dropped, Node 18+ required[^4]. |
| 6.0 | 2025-11 | Continued modernization on the ESM-only line[^8]. |

## References

[^1]: chaijs/chai — repository created 2011-12-07. https://github.com/chaijs/chai
[^2]: Core contributors listed in the project README (Keith Cirkel, James Garbutt, Kristján Oddsson). https://github.com/chaijs/chai#core-contributors
[^3]: Vitest docs, "Assert" / built-in `expect` API built on Chai. https://vitest.dev/api/expect.html
[^4]: Chai 5.0.0 release notes — ESM-only, Node 18+, UMD build removed. https://github.com/chaijs/chai/releases/tag/v5.0.0
[^5]: Chai's supporting packages: deep-eql, check-error, loupe, pathval. https://github.com/chaijs
[^6]: `dirty-chai` — plugin adding terminating parentheses to avoid no-op getter assertions. https://github.com/prodatakey/dirty-chai
[^7]: Chai 4.0.0 release notes. https://github.com/chaijs/chai/releases/tag/4.0.0
[^8]: Chai releases page. https://github.com/chaijs/chai/releases

## Tags

javascript, testing, assertion-library, bdd, tdd, mocha, esm, test-framework, node, browser
