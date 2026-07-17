# Tyrrrz/CliWrap

> A fluent, immutable .NET wrapper over `System.Diagnostics.Process` that makes launching child processes, piping their streams, and awaiting them a first-class async operation.

[GitHub repo](https://github.com/Tyrrrz/CliWrap) ·
[NuGet package](https://nuget.org/packages/CliWrap) ·
[License: MIT](https://github.com/Tyrrrz/CliWrap/blob/prime/License.txt)

## Overview

CliWrap is a single-purpose library: it wraps the .NET `Process` class so that running an external command becomes a configured, awaitable object instead of a pile of imperative event-handler and stream-draining boilerplate. It has been maintained by Oleksii Holub (Tyrrrz) since 2017[^1], and the API most people use today is the version 3 line, which was a ground-up rewrite around immutability and a composable piping model. As of 2026 it sits near 5k stars and is one of the most-downloaded process-execution packages on NuGet, with no external dependencies and targets down to .NET Standard 2.0 / .NET Framework 4.6.2[^2].

The problem it solves is real and specific: `Process` is notoriously easy to misuse. Reading stdout and stderr synchronously can deadlock if a buffer fills; `WaitForExit()` and async output events race; cancellation and cleanup of orphaned processes is manual. CliWrap's defining choice is to hide all of that behind a fluent builder whose terminal operation (`ExecuteAsync()`) drains both streams concurrently, wires up cancellation, and never leaves you half-configured. The tradeoff is that it is an opinionated abstraction: it throws on non-zero exit codes by default, and everything routes through its pipe model, so trivial "just run this and ignore output" calls carry a little more ceremony than a bare `Process.Start`.

CliWrap ships with a prominent political "terms of use" section in its README condemning Russia's invasion of Ukraine; the maintainer is Ukrainian and several of his projects carry the same banner. It is a statement, not a license term — the actual license is MIT and imposes no such condition[^3].

## Getting Started

```bash
dotnet add package CliWrap
```

```csharp
using CliWrap;
using CliWrap.Buffered;

// Run `git log --oneline`, capture stdout/stderr into memory.
var result = await Cli.Wrap("git")
    .WithArguments(["log", "--oneline", "-n", "5"])
    .WithWorkingDirectory("/path/to/repo")
    .ExecuteBufferedAsync();

Console.WriteLine(result.StandardOutput);
Console.WriteLine($"exit={result.ExitCode} took={result.RunTime}");
```

`ExecuteAsync()` (in the base namespace) runs the command with streams routed to a null sink and returns exit code and timing. `ExecuteBufferedAsync()` (in `CliWrap.Buffered`) additionally captures stdout/stderr as strings — convenient, but it holds all output in memory, so avoid it for commands that emit large or binary output.

## Architecture / How It Works

The core type is `Command`, an immutable value: every `WithXyz(...)` call returns a new instance rather than mutating in place. A configured command carries the target file path, an arguments string, working directory, environment variables, a validation policy, a resource policy, credentials, and three pipe endpoints (stdin source, stdout target, stderr target).

Piping is the conceptual center. Two abstractions do the work: `PipeSource` feeds the child's stdin, and `PipeTarget` consumes its stdout or stderr. Factory methods cover streams, files, byte arrays, strings, `StringBuilder`, and per-line delegates, plus `PipeTarget.Merge(...)` to fan one stream out to several targets. CliWrap overloads the `|` operator so commands compose like a shell pipeline — `input | Cli.Wrap("grep") | output`, or command-to-command chaining where one process's stdout becomes the next process's stdin. Under the hood the pipeline is executed with concurrent tasks draining each stream, which is what sidesteps the classic `Process` deadlock where a full stdout buffer stalls the child while the parent is blocked writing stdin.

There are four execution models over the same `Command`:

- **Result** — `ExecuteAsync()` returns a `CommandResult` (exit code, start/exit time, run time).
- **Buffered** — `ExecuteBufferedAsync()` adds captured stdout/stderr strings.
- **Async event stream** — `ListenAsync()` yields an `IAsyncEnumerable<CommandEvent>` (`StartedCommandEvent`, `StandardOutputCommandEvent`, `StandardErrorCommandEvent`, `ExitedCommandEvent`) so you can `await foreach` over output as it arrives.
- **Observable event stream** — `Observe()` exposes the same events as an `IObservable<CommandEvent>` for Rx consumers.

Validation is a policy, not an exception you opt into: by default `CommandResultValidation.ZeroExitCode` throws `CommandExecutionException` on a non-zero exit, carrying the command and exit code. `WithValidation(CommandResultValidation.None)` disables it, after which `result.IsSuccess`, an implicit `bool`, and equality against an `int` exit code let you branch without a try/catch.

## Production Notes

- **Non-zero exit throws by default.** Many CLIs (grep, diff, some linters) use non-zero exit codes as normal signals. If you forget `WithValidation(CommandResultValidation.None)`, those commands throw `CommandExecutionException` and look like failures. This is the single most common first-run surprise.
- **`ExecuteBufferedAsync()` buffers everything in memory.** For a command that streams gigabytes or writes binary, this will balloon memory. Use the plain `ExecuteAsync()` with an explicit `PipeTarget.ToStream(...)`/`ToFile(...)`, or the event-stream model, for large or unbounded output.
- **`PipeTarget.Null` does not open the stream at all.** As an optimization, piping to `Null` means the child's stdout/stderr is never redirected. This is normally equivalent to discarding, but a few programs behave differently when they detect no attached output stream[^4]; if that bites you, pipe explicitly to `PipeTarget.ToStream(Stream.Null)`.
- **Cancellation has two modes.** A single `CancellationToken` forcefully kills the process (and its children). CliWrap also supports graceful cancellation via interrupt signals through a separate token overload, letting the child clean up before a hard kill — useful for processes that must flush or release resources, but the target must actually handle the interrupt.
- **Argument escaping.** Prefer the array or builder overloads of `WithArguments(...)`, which escape each token for you. The raw-string overload requires you to escape and quote correctly yourself; getting it wrong is a source of both bugs and shell-injection-style vulnerabilities. Note CliWrap launches the executable directly (no shell), so shell features like globbing, `&&`, or redirection are not interpreted — you compose those with the pipe model or by invoking a shell explicitly.
- **Resource-policy and credential options are platform-dependent.** Priority, CPU affinity, and working-set limits, and most `WithCredentials` fields beyond username, are effectively Windows-only; they silently do less on Linux/macOS.

## When to Use / When Not

**Use when:**
- You call external tools (git, ffmpeg, docker, database CLIs) from .NET and want deadlock-safe stream handling.
- You need to compose pipelines, transform output line-by-line, or stream output events as they occur.
- You want cancellation, timing, and exit-code handling wired up correctly without writing it yourself.

**Avoid when:**
- You only need to fire-and-forget one process and never read its output — bare `Process.Start` is fewer concepts.
- You need genuine shell semantics (pipes, redirection, `&&`, environment expansion handled by a shell) — invoke the shell yourself; CliWrap does not parse a command line.
- You are in an environment where taking any dependency is discouraged (though the maintainer offers a source-internalizer, Binternal, for that case).

## Alternatives

- madelson/MedallionShell — the closest peer: fluent, cross-platform process/piping wrapper. Use it when you want a similar model with slightly different piping and shell ergonomics.
- adamralph/SimpleExec — minimal "run a command, throw on failure" helper. Use it in build scripts where you don't need piping or streaming.
- Cysharp/ProcessX — async, `IAsyncEnumerable`-first process execution. Use it when your main need is streaming stdout lines with minimal API surface.
- jamesmanning/RunProcessAsTask — a thin `Task`-returning wrapper over `Process`. Use it when you want async + captured output and nothing more.
- dotnet/runtime `System.Diagnostics.Process` — the built-in class. Use it directly when you want zero dependencies and accept handling stream draining and cancellation yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-03 | First release — a thin convenience wrapper over `Process`[^1]. |
| 3.0 | 2020 | Ground-up rewrite: immutable `Command`, fluent builder, `PipeSource`/`PipeTarget` piping model, `|` operators, buffered + event-stream execution models. |
| 3.x | 2021–2022 | Graceful cancellation via interrupt signals; `ArgumentsBuilder` refinements; broader target frameworks. |
| 3.x | 2023–2024 | `WithResourcePolicy`, array-based `WithArguments`, updated .NET targets. |
| active | 2026 | Maintained; last push 2026-07 on the `prime` default branch[^2]. |

## References

[^1]: CliWrap repository, created 2017-03-30. https://github.com/Tyrrrz/CliWrap
[^2]: GitHub API metadata for Tyrrrz/CliWrap (stars ~4,992; MIT; C#; default branch `prime`; last push 2026-07-04), retrieved 2026-07. https://github.com/Tyrrrz/CliWrap
[^3]: CliWrap README, "Terms of use" section, and License.txt (MIT). https://github.com/Tyrrrz/CliWrap/blob/prime/Readme.md
[^4]: CliWrap issue #145 — behavior of `PipeTarget.Null` vs an explicit null stream. https://github.com/Tyrrrz/CliWrap/issues/145

## Tags

csharp, dotnet, process, cli, command-line, subprocess, piping, async, stdio, wrapper-library, nuget
