# google/python-fire

> Turns any Python object — function, class, module, dict — into a command-line interface by introspection, with zero CLI-definition code.

[GitHub repo](https://github.com/google/python-fire) ·
[License: Apache-2.0](https://github.com/google/python-fire/blob/master/LICENSE)

## Overview

Python Fire is a library that generates a command-line interface from an arbitrary Python object[^1]. You call `fire.Fire(component)` and Fire inspects the component at runtime — its functions, arguments, class methods, dict keys — and maps `argv` onto them. There is no CLI schema to declare: the CLI *is* the object's shape. Google open-sourced it in March 2017[^2] after internal use for turning existing code into throwaway tools.

The defining tradeoff is **magic versus predictability**. Because the CLI is derived by reflection, adding a keyword argument to a function silently adds a flag, and renaming a method silently renames a subcommand. That is exactly what makes Fire fast for exploration and debugging, and exactly what makes it a poor fit for a stable, user-facing CLI where the interface is a contract you want to version deliberately. Fire's own tagline — a CLI for "absolutely any Python object" — is both the feature and the warning.

Fire is widely used inside ML and research code (it is a common pattern in TensorFlow-era Google repos and appears in many training-script entry points) precisely because it removes boilerplate from scripts whose arguments change constantly. It is comparatively rare in polished distributed CLIs, where Click or Typer dominate. The project is maintained but low-churn: releases are infrequent and the API surface has been stable for years[^3].

## Getting Started

```bash
pip install fire
# or: conda install fire -c conda-forge
```

```python
import fire

def hello(name="World"):
    return f"Hello {name}!"

if __name__ == "__main__":
    fire.Fire(hello)
```

```bash
python hello.py                # Hello World!
python hello.py --name=David   # Hello David!
python hello.py --help         # introspected usage
```

Calling Fire on a class exposes its methods as subcommands, positional or by flag:

```python
import fire

class Calculator:
    def double(self, number):
        return 2 * number

if __name__ == "__main__":
    fire.Fire(Calculator)
```

```bash
python calc.py double 10          # 20
python calc.py double --number=15 # 30
```

## Architecture / How It Works

Fire's core is a **recursive trace over a Python object**, driven by the `inspect` module[^1]:

1. **Introspect the component.** Fire examines the object's type. A function becomes a callable target; a class is instantiated then traversed; a dict/list/tuple exposes its keys/indices as subcommands; a module exposes its members; a plain object exposes attributes and methods.
2. **Consume `argv` left to right.** Each token either selects a member (attribute access / method / key — a *trace step*) or supplies an argument. Fire keeps a `FireTrace` recording each step, which is what powers the `--trace` flag and its error messages.
3. **Bind arguments.** Positional tokens fill positional parameters; `--name=value` tokens fill keyword parameters. Fire uses signature introspection (`inspect.signature`) to know what fits.
4. **Parse argument values.** Each raw string is run through a literal parser (built on `ast.literal_eval`): `10` becomes an `int`, `[1,2]` a `list`, `True` a `bool`, and anything that fails to parse stays a `str`.
5. **Terminate or continue.** If the current value is callable and arguments remain, Fire calls it and recurses on the result. When it runs out of tokens it prints the final result (via `str`/`repr`) or, for help, renders introspected usage.

Fire-level flags (`--help`, `--interactive`, `--trace`, `--completion`, `--separator`, `--verbose`) are separated from your component's arguments by an isolated `--`. Everything before the `--` belongs to the traversal; everything after configures Fire itself. This `--` boundary is the single most misunderstood part of the model.

Help and the interactive REPL are also built on the same introspection: `--help` renders the discovered signature and docstrings; `-- --interactive` drops into a Python/IPython shell with the traced component and locals preloaded.

## Production Notes

**Argument type coercion is a footgun.** Because values are parsed as Python literals, a version string like `1.0` arrives as a `float`, a ZIP code `01234` may be reinterpreted, and a name that happens to be `True`/`None`/`10` arrives as a `bool`/`NoneType`/`int` rather than the string you expected[^4]. Quoting rules interact with the shell, so `--x='"10"'` is sometimes needed to force a string. There is no declarative type schema to override this per-argument; you validate inside the function.

**Help output is paged by default.** Fire sends `--help` through a pager (`less`), which is disorienting in CI, notebooks, and logs, and can appear to "hang." Set `PAGER=cat` or pipe the command to disable paging. This bites first-time users constantly.

**Reflection exposes more than you intend.** Firing a large object makes *every* public attribute and method reachable from the command line, including ones you did not mean to be a CLI surface. Fire a narrow, purpose-built object (or a dict of explicit entrypoints), not a fat application object, if untrusted input can reach it.

**Not a contract.** Since the interface is derived, ordinary refactors (renaming a method, reordering parameters, adding a kwarg) are breaking CLI changes with no compiler or type checker to catch them. There is no built-in mechanism for deprecation, hidden commands, mutually exclusive flags, or custom usage strings. For a CLI shipped to external users, this lack of an explicit, reviewable interface definition is the main reason teams migrate off Fire.

**Shell completion exists but is second-class.** `-- --completion` generates a bash/zsh script by introspection; it is serviceable but not the polished, dynamically-updated completion that Click/Typer provide.

**Stability.** The upside of low churn: Fire rarely breaks your build on upgrade, dependencies are minimal (`six`, `termcolor`), and it runs on current Python 3. The downside: newer ergonomics (type-hint-driven parsing, rich errors) are not coming — the design is settled.

## When to Use / When Not

**Use when:**
- You want to expose existing functions/classes as a CLI with essentially no extra code.
- You are debugging, exploring an unfamiliar codebase, or scripting internal/ML training entrypoints whose arguments churn constantly.
- The CLI is for you or your team, and predictability of the interface matters less than speed of creation.

**Avoid when:**
- You are shipping a user-facing CLI where the argument surface is a versioned contract.
- You need strict per-argument typing, validation, custom error messages, or first-class completion.
- Untrusted input reaches the CLI and you cannot tightly scope the exposed object.

## Alternatives

- pallets/click — decorator-based, explicit CLI definitions; the default choice for a production, user-facing CLI.
- fastapi/typer — type-hint-driven CLIs built on Click; use when you want Fire-like brevity but explicit, validated arguments.
- python/cpython (argparse, stdlib) — no dependency, verbose but fully explicit; use when you cannot add a dependency.
- docopt/docopt — define the interface from the usage docstring; use when you want the help text to *be* the spec.
- google/python-fire is itself the right pick only when zero-definition introspection is the point.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2017-03 | Initial open-source release by Google[^2]. |
| 0.2.0 | 2019 | Python 3 focus, trace/help improvements. |
| 0.3.0 | 2020 | Continued Python 3 support, bug fixes. |
| 0.4.0 | 2020 | Maintenance release. |
| 0.5.0 | 2023 | Maintenance and compatibility updates[^3]. |
| 0.6.0 | 2024 | Maintenance release. |
| 0.7.0 | 2024 | Latest line; API stable, minimal churn[^3]. |

## References

[^1]: Python Fire README and guide — `fire.Fire()` generates a CLI from any object via introspection. https://github.com/google/python-fire/blob/master/docs/guide.md
[^2]: David Bieber, "Python Fire, a library for automatically generating command line interfaces" — Google Developers/Open Source blog, 2017-03-02. https://opensource.googleblog.com/2017/03/python-fire-command-line.html
[^3]: Python Fire releases and changelog. https://github.com/google/python-fire/releases
[^4]: Python Fire — "Using a Fire CLI": argument parsing treats values as Python literals. https://github.com/google/python-fire/blob/master/docs/using-cli.md

## Tags

python, cli, command-line, introspection, reflection, developer-tools, google, argument-parsing, scripting, rapid-prototyping
