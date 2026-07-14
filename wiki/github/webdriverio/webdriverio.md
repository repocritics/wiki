# webdriverio/webdriverio

> Node.js browser and mobile automation framework built on the W3C WebDriver and WebDriver BiDi protocols, with a plugin-based test runner.

[GitHub repo](https://github.com/webdriverio/webdriverio) ·
[Official website](http://webdriver.io) ·
[License: MIT](https://github.com/webdriverio/webdriverio/blob/main/LICENSE)

## Overview

WebdriverIO is a test automation framework for Node.js that drives real browsers and mobile devices. Unlike tools that ship their own automation engine, it speaks standardized wire protocols — W3C WebDriver (the "classic" HTTP protocol behind Selenium) and, increasingly, WebDriver BiDi (a bidirectional WebSocket protocol) — plus Appium for native mobile[^1]. This protocol-first stance is its defining characteristic: it can drive anything that implements WebDriver, from local Chrome/Firefox to a Sauce Labs or BrowserStack cloud grid to an Appium session on a physical phone, through one command API.

The project is really two things stacked. The lower layer (`webdriver`, `webdriverio` packages) is a protocol binding plus a fluent element/browser command API. The upper layer (`@wdio/cli` and its ecosystem of services, reporters, framework adapters, and runners) is a full test runner configured through a `wdio.conf.ts` file. You can use the binding standalone as a scripting library, or adopt the runner for parallelized suites with Mocha, Jasmine, or Cucumber.

Its defining tension is protocol fidelity versus developer experience. Because it stays close to WebDriver, it inherits both the breadth (real cross-browser, real mobile, real grids) and the friction (HTTP round-trips, driver/browser version coupling, more configuration surface) of that world. Competitors like Playwright and Cypress trade some of that breadth for a tighter, faster, single-vendor loop. WebdriverIO's answer has been to adopt WebDriver BiDi as it matures, closing much of the capability gap (events, network interception, console logs) without abandoning the standard.

## Getting Started

```bash
npm init wdio@latest ./
# interactive wizard: runner, framework, reporters, services, drivers
```

```ts
// wdio.conf.ts (excerpt)
export const config: WebdriverIO.Config = {
  runner: 'local',
  specs: ['./test/specs/**/*.ts'],
  capabilities: [{ browserName: 'chrome' }],
  framework: 'mocha',
  reporters: ['spec'],
  services: [],
}
```

```ts
// test/specs/example.e2e.ts
import { browser, $ } from '@wdio/globals'

describe('search', () => {
  it('finds a result', async () => {
    await browser.url('https://duckduckgo.com/')
    await $('input[name="q"]').setValue('WebdriverIO')
    await browser.keys('Enter')
    await expect($('#links')).toBeDisplayed()  // auto-retries until timeout
  })
})
```

## Architecture / How It Works

The core is a thin protocol client. `@wdio/protocols` describes every WebDriver, BiDi, Appium, and vendor command as data; the `webdriver` package turns those descriptions into methods that issue HTTP requests (classic) or WebSocket messages (BiDi) to a driver. `webdriverio` wraps that client with the ergonomic surface most users see: `browser`, element references from `$`/`$$`, chainable commands, and automatic waiting — element lookups retry until `waitforTimeout` elapses rather than failing on first miss.

The test runner is a separate concern built for parallelism. `@wdio/cli` reads `wdio.conf`, then `@wdio/local-runner` spawns each spec file in its own Node worker process (up to `maxInstances`), so browser sessions run concurrently and isolated. Around that sit four plugin families, all resolved from config: **framework adapters** (`@wdio/mocha-framework`, `-jasmine-`, `-cucumber-`) that own the test lifecycle; **reporters** (spec, dot, junit, allure, ...) that consume the event stream; **services** (`@wdio/sauce-service`, `-appium-`, `-browserstack-`, ...) that hook lifecycle events to start drivers, upload results, or wire up grids; and **runners** (`@wdio/local-runner`, `@wdio/browser-runner`). The `browser-runner` is notable — it executes unit/component tests inside a real browser via Vite, so the same tool covers e2e and component testing[^2].

Selector handling is broader than most: CSS and XPath, plus strategies for shadow DOM (deep selectors), accessibility name, React component props, and text matching. Driver management is built in as of v8 — WebdriverIO downloads and starts the correct `chromedriver`/`geckodriver` itself, replacing the old external `selenium-standalone` dependency[^3].

## Production Notes

**Driver/browser version coupling is the classic footgun.** Chrome auto-updates; a mismatched pinned `chromedriver` breaks sessions. Built-in driver management (v8+) mostly solves this by resolving the driver to the installed browser, but self-managed or air-gapped CI still has to pin and mirror both halves deliberately.

**WebDriver classic is chatty.** Every command is an HTTP round-trip, so latency to the driver (or a remote grid) dominates wall-clock time on large suites. WebDriver BiDi, which v9 makes the default where supported, moves to a persistent WebSocket and adds event-driven capabilities (network interception, console/log events, `mock`) that previously required the deprecated `devtools` protocol path[^4]. Not every capability is available on every driver via BiDi yet — expect a mixed classic/BiDi reality for now.

**The v8 ESM migration was disruptive.** v8 moved the codebase to ES modules and dropped older Node versions[^5]. Projects with CommonJS configs, `require`-based helpers, or ESM-hostile transitive deps hit real friction; TypeScript `module`/`moduleResolution` settings and `ts-node`/`tsx` loader choices all matter. Synchronous command mode (the old `@wdio/sync`, built on `node-fibers`) was removed — all commands are `async`/`await`, since fibers no longer work on modern Node[^5].

**Flakiness management is on you.** Auto-waiting for element presence helps, but network, animation, and app-state timing still produce flakes. Standard mitigations: explicit `waitUntil` conditions over fixed sleeps, retry at the test level (`mochaOpts.retries` or spec-level `this.retries`), and isolating shared-state suites. Parallel workers (`maxInstances`) speed suites up but multiply resource use — under-provisioned CI causes session timeouts that look like test failures.

**Cloud and mobile are first-class but add config.** Grids (Sauce, BrowserStack, TestingBot) and Appium are wired through services and capabilities rather than the core, so most integration issues are configuration, credentials, and capability-matrix problems, not framework bugs.

## When to Use / When Not

**Use when:**
- You need genuine cross-browser coverage (Chrome, Firefox, Safari/WebKit, Edge) on real engines, not just Chromium.
- Mobile matters — Appium integration gives native iOS/Android automation through the same API.
- You run on a cloud grid (Sauce Labs, BrowserStack) or your own Selenium grid.
- You want a plugin architecture and BDD support (Cucumber/Mocha/Jasmine) with parallel workers.

**Avoid when:**
- You want the fastest possible Chromium-only inner loop with minimal config — Playwright or Cypress deliver a tighter DX.
- Your testing is API/back-end only — a browser automation framework is the wrong tool.
- Your team wants a single bundled binary with no driver/protocol surface to reason about.
- You need in-browser time-travel debugging as a primary workflow (Cypress's niche).

## Alternatives

- microsoft/playwright — use instead when you want a modern single-vendor tool with bundled browsers, auto-wait, tracing, and no separate driver management.
- cypress-io/cypress — use instead for an in-browser runner with time-travel debugging on a mostly-Chromium web app.
- SeleniumHQ/selenium — use instead when you need the broadest language bindings and raw WebDriver/grid control across ecosystems.
- nightwatchjs/nightwatch — use instead for an integrated Selenium-based Node runner with a simpler, more prescriptive config.
- appium/appium — use directly when the target is mobile-only; WebdriverIO wraps Appium rather than replacing it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 4.x | 2016–2018 | Selenium-standalone era; synchronous command mode via fibers. |
| 5.0 | 2018 | Major rewrite: monorepo, W3C WebDriver focus, new test runner architecture[^3]. |
| 6.0 | 2020 | Core rewritten in TypeScript; automatic driver handling groundwork. |
| 7.0 | 2021 | Updated TypeScript support and DevTools/automation protocol improvements. |
| 8.0 | 2022-12 | ESM migration, sync mode removed, built-in driver management, Node floor raised[^5]. |
| 9.0 | 2024 | WebDriver BiDi as the default protocol where supported; expanded BiDi commands[^4]. |

## References

[^1]: WebdriverIO README and homepage — protocols supported (WebDriver, WebDriver BiDi, Appium). https://webdriver.io/
[^2]: WebdriverIO docs, "Component Testing" (browser runner via Vite). https://webdriver.io/docs/component-testing
[^3]: WebdriverIO blog, "WebdriverIO v5 released" — architecture rewrite. https://webdriver.io/blog/
[^4]: WebdriverIO docs, "WebDriver BiDi" and v9 announcement. https://webdriver.io/docs/api/webdriverBidi
[^5]: WebdriverIO blog, "WebdriverIO v8 released" — ESM migration and removal of synchronous mode. https://webdriver.io/blog/2023/01/03/webdriverio-v8-released

## Tags

testing, browser-automation, webdriver, webdriver-bidi, appium, e2e-testing, typescript, nodejs, test-runner, mobile-testing
