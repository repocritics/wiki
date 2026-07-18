# pcapriotti/optparse-applicative

> The de facto standard command-line option parser for Haskell, built on an
> applicative interface that trades monadic flexibility for static
> introspection: help text, usage lines, and shell completions for free.

[GitHub repo](https://github.com/pcapriotti/optparse-applicative) ·
[Hackage](https://hackage.haskell.org/package/optparse-applicative) ·
License: BSD-3-Clause

## Overview

optparse-applicative is a Haskell library for declaring command-line
interfaces as values of type `Parser a`, composed with the standard
`Applicative` and `Alternative` type classes. Started by Paolo Capriotti in
2012, with day-to-day maintenance largely carried by Huw Campbell in later
years, it has become the default answer to "how do I parse CLI arguments in
Haskell" — it is one of the most-depended-upon packages on Hackage and powers
prominent tools such as stack[^1]. The GitHub star count (958) badly
undercounts its reach, since Haskell developers consume it via Hackage, not
GitHub.

The defining design decision is the applicative (rather than monadic)
interface. Because a `Parser a` is a static structure — options cannot depend
on the values of other options — the library can inspect the whole parser
without running it. That is what makes automatic usage lines, a full help
screen, error reporting, and context-sensitive bash/zsh/fish completion
scripts possible[^2]. The cost is real: you cannot express "if `--format=json`
was given, require `--schema`" inside the parser itself; such cross-option
constraints must be validated after parsing. This tension — rich static
tooling vs. no inter-option dependencies — is the library in one sentence.

The project is alive but deliberately slow-moving: created 2012, latest
release 0.19.0 (June 2025), last push June 2026, 33 open issues. For a
13-year-old library at the center of an ecosystem, that cadence reads as
"mature and conservative", not "abandoned".

## Getting Started

Add to your `.cabal` file:

```
build-depends: optparse-applicative >= 0.18
```

Minimal parser (from the project README, abridged):

```haskell
import Options.Applicative

data Sample = Sample
  { hello :: String
  , quiet :: Bool }

sample :: Parser Sample
sample = Sample
  <$> strOption (long "hello" <> metavar "TARGET" <> help "Greeting target")
  <*> switch    (long "quiet" <> short 'q' <> help "Whether to be quiet")

main :: IO ()
main = do
  opts <- execParser $ info (sample <**> helper)
            (progDesc "Print a greeting for TARGET")
  print (hello opts)
```

Running with `--help` prints a generated help screen; missing mandatory
options produce an error plus a usage summary automatically.

## Architecture / How It Works

`Parser` is a small algebraic structure — essentially a free-applicative-style
tree over primitive option specifications, with an `Alternative` layer for
choice[^2]. Interpretation happens twice: once symbolically (walking the tree
to derive usage text, help, and completion metadata) and once concretely
(matching real `argv` against it). Because options in an applicative chain are
order-independent on the command line, the parser is effectively a permutation
parser: `--file x -q` and `-q --file x` yield the same result.

The user-facing surface is the *builder* API: primitives (`strOption`,
`option`, `switch`, `flag'`, `argument`, `subparser`) each take a *modifier*
argument composed with the `Semigroup` operator `<>` (`long`, `short`,
`metavar`, `help`, `value`, `showDefault`, ...). Git-style subcommands are
just another parser primitive (`subparser` / `hsubparser` with `command`),
which is why nested command trees compose the same way flat options do.

For cases where applicative style gets awkward (many fields, interleaved
transformations), the library offers an arrow interface and works with GHC's
`ApplicativeDo` — with the caveat that the do-block must actually desugar to
applicative operations, or it will not compile against `Parser`, which has no
`Monad` instance.

Help rendering was rebuilt in 0.18 on top of the `prettyprinter` library,
replacing the deprecated `ansi-wl-pprint`, and gained styled/colored output;
this was a breaking change for anyone customizing help documents[^3].

## Production Notes

- **`<**> helper` is a canonical footgun.** The `helper` combinator adds
  `--help`; forget it and your tool has no help flag at all. It must also be
  applied with `<**>` (reversed apply) after the parser — writing `helper <*>`
  or misplacing it is a classic first-hour mistake.
- **`option auto` uses `Read`.** Parsing a `String` via `auto` requires the
  user to type shell-escaped Haskell-style quotes, and `Read` failures produce
  opaque "cannot parse value" errors. Use `strOption` for strings and write
  custom `ReadM` readers (`eitherReader`) for anything needing decent error
  messages.
- **Defaults change UX more than you expect.** Out of the box, an invalid
  invocation prints only the error and usage line. Most polished tools run
  `customExecParser (prefs $ showHelpOnError <> showHelpOnEmpty)` and consider
  `disambiguate` (unambiguous prefix matching), all off by default.
- **Cross-option constraints live outside the parser.** Requiring option B
  when option A is set means post-parse validation in your own code, with
  error messages that will not match the library's formatting. Sum types plus
  `<|>` cover many cases (mutually exclusive modes) but not conditional
  requirements.
- **Completion needs packaging work.** Completion scripts are generated at
  runtime via hidden flags (`--bash-completion-script` etc.); they only help
  users if your package's install step actually registers them with the shell.
- **The 0.17 → 0.18 upgrade** forced the `prettyprinter` migration through
  downstream dependency trees; pins on `ansi-wl-pprint` had to move[^3].
  Routine GHC-release version-bound bumps are the main other churn — the
  library itself breaks rarely.
- **Performance is a non-issue** for CLI workloads: the parser is interpreted
  per invocation, and even large subcommand trees parse in microseconds
  relative to process startup.

## When to Use / When Not

**Use when:**
- You are writing any nontrivial Haskell CLI — this is the ecosystem default,
  and the one alternatives measure themselves against.
- You want generated help, usage, and shell completions guaranteed to stay in
  sync with the accepted options.
- You have git-style subcommands; `subparser` composes them cleanly.

**Avoid when:**
- Your interface has heavy inter-option dependencies; you will fight the
  applicative structure and end up validating by hand anyway.
- The tool is tiny and dependency-count-sensitive: `System.Console.GetOpt`
  ships with base.
- You want parsers derived from record types with zero ceremony —
  optparse-generic trades control for brevity.

## Alternatives

- ndmitchell/cmdargs — annotation-based, record-driven definitions; use when
  you want terse impure-style declarations and accept less active upkeep.
- Gabriella439/optparse-generic — derives parsers from types via Generics (on
  top of optparse-applicative); use when boilerplate matters more than
  fine-grained help-text control.
- fpco/optparse-simple — thin convenience wrapper (e.g. version-from-git
  helpers); use alongside, not instead of, the core library.
- System.Console.GetOpt (in GHC's base library) — zero extra dependencies; use
  for trivial single-purpose tools.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-05 | Repository created by Paolo Capriotti. |
| 0.14.0 | 2017-06 | `Semigroup` era; long-lived stable line (0.14.x through 2019). |
| 0.15.0 | 2019-07 | New builders and reader utilities. |
| 0.16.0 | 2020-08 | Continued API refinement; wide GHC compatibility. |
| 0.17.0 | 2022-02 | Maintenance release line. |
| 0.18.0 | 2023-05 | Switch from ansi-wl-pprint to prettyprinter; styled help[^3]. |
| 0.19.0 | 2025-06 | Current release line. |

Release dates above are GitHub release timestamps; see the changelog for
per-version detail[^3].

## References

[^1]: Hackage package page (reverse dependencies, maintainer listing).
    https://hackage.haskell.org/package/optparse-applicative
[^2]: Project README, "How it works" and "Applicative" sections.
    https://github.com/pcapriotti/optparse-applicative#how-it-works
[^3]: Changelog.
    https://github.com/pcapriotti/optparse-applicative/blob/master/CHANGELOG.md

## Tags

haskell, cli, option-parsing, applicative, parser-combinators, command-line, shell-completion, library, developer-tools
