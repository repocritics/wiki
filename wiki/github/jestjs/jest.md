# jestjs/jest

The JavaScript testing framework from Meta — zero-config in most cases, snapshot testing as a defining feature, broadly adopted across React + Node ecosystems.

## What it is

A test framework that combines a test runner, assertion library, mocking primitives, and a coverage reporter into one tool. Originated at Meta for testing React + Node code; donated to the OpenJS Foundation in 2022. The defining differentiator was zero-config + snapshot testing (capture component output, compare across runs). Vitest has supplanted Jest in many newer projects but Jest remains the most-installed JS test tool.

## Key features

- Zero-config for most JS / TS projects.
- Snapshot testing — capture serialized output, fail on diff.
- Mocking: jest.fn(), jest.mock(), automatic mocking of module imports.
- Parallel test execution across files.
- Coverage reporting via Istanbul.
- Watch mode with intelligent re-runs.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Node.js runtime.
- npm packages: `jest`, `@jest/*`, `jest-environment-*`.

## When to reach for it

- You're maintaining an existing codebase that uses Jest (most React / older Node projects).
- You want zero-config with React.
- You need rich mocking primitives.

## When *not* to reach for it

- You're starting a new Vite-based project — Vitest fits more naturally.
- You want max speed — Vitest, Bun's test runner, and node:test are faster.

## Maturity signal

45k stars, 7k forks, MIT, actively maintained under the OpenJS Foundation. The 230 open-issues count is low and signals tight maintenance.

## Alternatives

- Vitest — Vite-flavored Jest-compatible alternative; the modern default.
- Mocha + Chai + Sinon — traditional JS test stack.
- node:test — Node 20+ built-in test runner.
- Bun test — Bun's built-in test runner.

## Tags

typescript, javascript, testing, framework, jest, mit-license, snapshot-testing
