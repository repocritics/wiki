# elm/compiler

> A statically typed functional language that compiles to JavaScript and promises no runtime exceptions — at the cost of tightly controlled interop and a near-frozen release cadence.

[GitHub repo](https://github.com/elm/compiler) ·
[Official website](https://elm-lang.org/) ·
[License: BSD-3-Clause](https://github.com/elm/compiler/blob/main/LICENSE)

## Overview

Elm is a pure functional language for building web frontends, compiling to JavaScript. It began as Evan Czaplicki's 2012 thesis and grew into a full language with its own compiler (written in Haskell), package manager, and standard architecture[^1]. Its defining pitch is reliability: a sound-by-design type system with no `null`/`undefined`, exhaustive pattern matching, and managed effects, together yielding the marketing claim of "no runtime exceptions in practice." The compiler's famously readable error messages set a bar that later influenced Rust, Scala, and TypeScript diagnostics[^2].

The defining tension is governance and pace. Elm is effectively a single-maintainer language, and the last release, 0.19.1, shipped in October 2019[^3]. There has been no tagged compiler release in years despite ongoing repository activity. For teams that value stability this reads as "finished"; for those expecting active evolution it reads as "stalled." The 0.19 release also removed the community's ability to write custom "native"/kernel JavaScript modules and restricted publishing of such packages to the core team, a decision that fractured parts of the ecosystem and remains the most-cited criticism of the project[^4].

Elm is best understood as a language plus an enforced application shape (The Elm Architecture) plus a curated package registry — a coherent, opinionated whole rather than a library you drop into an existing stack.

## Getting Started

```bash
# Install via the official installer (npm wrapper, or platform binaries)
npm install -g elm
# or download the binary: https://guide.elm-lang.org/install/elm.html

elm init            # creates elm.json
elm reactor         # dev server at http://localhost:8000
elm make src/Main.elm --output=main.js
```

```elm
module Main exposing (main)

import Browser
import Html exposing (Html, button, div, text)
import Html.Events exposing (onClick)

type alias Model = Int
type Msg = Increment | Decrement

update : Msg -> Model -> Model
update msg model =
    case msg of
        Increment -> model + 1
        Decrement -> model - 1

view : Model -> Html Msg
view model =
    div []
        [ button [ onClick Decrement ] [ text "-" ]
        , text (String.fromInt model)
        , button [ onClick Increment ] [ text "+" ]
        ]

main : Program () Model Msg
main =
    Browser.sandbox { init = 0, update = update, view = view }
```

## Architecture / How It Works

**The Elm Architecture (TEA).** Every Elm app is a `Model` (immutable state), a `Msg` type (every event that can occur), an `update` reducer, and a `view`. This unidirectional data flow directly inspired Redux[^5]. Side effects are not run inline; `update` returns `Cmd Msg` values that the runtime executes, feeding results back as new messages. External events arrive through `Sub Msg` subscriptions. The 0.17 release (2016) removed the original FRP "signals" model in favor of this commands/subscriptions design[^6].

**Compiler.** `elm make` is a whole-program Haskell compiler that parses, type-infers (Hindley-Milner with extensions), checks exhaustiveness, and emits a single JavaScript bundle. Because the type system has no escape hatches (`any`, casts, exceptions), the compiler can perform aggressive dead-code elimination — unused package functions are dropped, so bundle size tracks what you actually call rather than what you depend on.

**Interop is deliberately narrow.** Elm code cannot call JavaScript directly. The three sanctioned bridges are: **flags** (data passed in at startup), **ports** (typed, asynchronous message channels between Elm and JS), and **custom elements** (wrapping JS-driven DOM). There is no synchronous FFI. Since 0.19, only `elm/*` and `elm-explorations/*` packages may contain kernel JS, so a third-party library that needs a browser API Elm doesn't expose cannot be published as native code[^4].

**Enforced semantic versioning.** The registry compares a package's public API across versions; `elm diff` computes whether a change is patch/minor/major and the tooling refuses a version number that understates the actual API change[^7]. This is a genuinely unusual guarantee — a `MAJOR` bump provably means a breaking API change.

## Production Notes

- **The language is frozen in practice.** Planning around this is a strategic choice, not a temporary one. New browser APIs reach Elm only through ports or custom elements, never new core packages, because the core is closed to outside kernel code. Budget for glue JavaScript on anything cutting-edge.
- **Ports are asynchronous and untyped at the boundary.** Data crossing a port is decoded with `Json.Decode`; a shape mismatch fails the decoder at runtime (handled as a `Result`), which is the main place the "no runtime exceptions" claim leaks — bad JSON from JS is a runtime failure you must decode defensively.
- **No `debugger`-friendly stack traces.** Compiled output is minified functional JS; debugging usually happens through the time-travel debugger (`elm reactor` / `--debug`) rather than browser stack traces.
- **SSR/SEO story is thin.** Elm targets the client. Server-side rendering exists only via community shims; there is no first-party SSR, streaming, or metaframework.
- **Ecosystem gaps must be bridged manually.** Common needs (rich text editors, charting, mapping, WebGL beyond `elm-explorations/webgl`) frequently require wrapping a JS library behind a custom element or port rather than an Elm package.
- **Upgrades are rare but large.** The 0.18→0.19 migration removed user-defined operators, tightened interop, and broke many packages; because releases are infrequent, when one lands the churn is concentrated. There is no 1.0.

## When to Use / When Not

**Use when:**
- You want maximum frontend reliability and are willing to adopt an opinionated architecture wholesale.
- Your app is a long-lived client-side SPA where "compiles = works" and rare-but-clean upgrades are assets, not liabilities.
- Your team values compiler-enforced correctness and readable errors over library breadth.

**Avoid when:**
- You need deep, synchronous, or ad-hoc JavaScript interop, or you depend on niche browser APIs.
- You need server-side rendering, a metaframework, or a large third-party component ecosystem.
- You require an actively evolving language with predictable release cadence and a broad maintainer bench — Elm's bus factor and pace are real risks.

## Alternatives

- purescript/purescript — Haskell-like language to JS with a far richer type system and unrestricted FFI; use when you want Elm's rigor without the interop lockdown and are willing to assemble your own architecture.
- rescript-lang/rescript — OCaml-derived, pragmatic first-class JS interop and TypeScript-adjacent output; use when JS ecosystem access matters more than purity.
- gleam-lang/gleam — statically typed functional language targeting Erlang and JS; use when you want similar type-safety with a more active, less centralized community.
- fable-compiler/Fable — F# to JS; use when you want a mature .NET language and ecosystem on the frontend.
- clojure/clojurescript — dynamically typed Lisp to JS with strong React interop (Reagent/re-frame); use when you prefer REPL-driven dynamism and the JS ecosystem over static guarantees.

## History

| Version | Date | Notes |
|---------|------|-------|
| thesis  | 2012 | Original Elm, FRP-based, from Evan Czaplicki's thesis[^1]. |
| 0.17    | 2016-05 | Removed FRP signals; introduced commands/subscriptions[^6]. |
| 0.18    | 2016-11 | Refinements to TEA and tooling. |
| 0.19    | 2018-08 | Asset-size focus, aggressive dead-code elimination, restricted kernel/native code to core[^4]. |
| 0.19.1  | 2019-10 | Latest release; bug fixes and compiler stability[^3]. |

## References

[^1]: Evan Czaplicki, "Elm: Concurrent FRP for Functional GUIs" (senior thesis, 2012). https://elm-lang.org/assets/papers/concurrent-frp.pdf
[^2]: "Compiler Errors for Humans" — Elm blog on diagnostic design. https://elm-lang.org/news/compiler-errors-for-humans
[^3]: elm/compiler 0.19.1 release. https://github.com/elm/compiler/releases/tag/0.19.1
[^4]: "elm/compiler #1889 — hardcoded restriction on native/kernel modules," and community discussion of the 0.19 policy. https://github.com/elm/compiler/issues/1889
[^5]: Redux docs crediting The Elm Architecture as prior art. https://redux.js.org/understanding/history-and-design/prior-art
[^6]: "A Farewell to FRP" — Elm 0.17 announcement. https://elm-lang.org/news/farewell-to-frp
[^7]: "Elm's semantic versioning enforcement" — Guide / package publishing. https://guide.elm-lang.org/publishing/

## Tags

haskell, elm, compiler, functional-programming, frontend, javascript, web, spa, type-inference, elm-architecture, static-typing
