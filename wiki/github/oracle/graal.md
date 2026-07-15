# oracle/graal

> GraalVM — a JDK with an advanced JIT compiler, ahead-of-time native compilation, and a polyglot language-implementation framework, all written in Java.

[GitHub repo](https://github.com/oracle/graal) ·
[Official website](https://www.graalvm.org) ·
[License: GPL-2.0 with Classpath Exception (multi-license)](https://github.com/oracle/graal/blob/master/LICENSE)

## Overview

GraalVM is Oracle Labs' long-running compiler research project turned production JDK[^1]. The `oracle/graal` repository is the monorepo for several distinct things that share a code base: the **Graal compiler** (a JIT compiler for HotSpot written in Java), **Native Image** (ahead-of-time compilation of Java programs to standalone native executables via the SubstrateVM runtime), and **Truffle** (a framework for building high-performance language interpreters). Languages such as GraalJS, GraalPy, TruffleRuby, GraalWasm, Sulong (LLVM bitcode), and Espresso (Java bytecode on Truffle) are built on it, several living in sibling repositories.

The defining idea is *partial evaluation*: a Truffle interpreter, plus the program it is interpreting, can be specialized by the Graal compiler into machine code as if a bespoke compiler had been written for that language. The same compiler backend serves the JIT, the AOT `native-image` tool, and every guest language. This is the project's intellectual strength and its practical burden — one code base carries JIT, AOT, and polyglot concerns simultaneously.

The most consequential distinction for users is **GraalVM Community Edition vs. Oracle GraalVM**[^2]. This repository is the source of the Community Edition (GPLv2+CE). Oracle GraalVM is a separate distribution under Oracle's Free Terms and Conditions license that adds features not in this repo — notably G1 GC and profile-guided optimization (PGO) for Native Image, and additional compiler optimizations. Benchmarks quoted for "GraalVM" often mean Oracle GraalVM; the open-source build here can be materially slower and heavier.

## Getting Started

Install via SDKMAN (Community Edition build):

```bash
sdk install java 21-graalce
```

Or download a build from graalvm.org. Then compile a program to a native executable:

```java
// Hello.java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello from a native binary");
    }
}
```

```bash
javac Hello.java
native-image Hello           # produces ./hello — no JVM needed to run it
./hello                       # starts in ~milliseconds, small RSS
```

For real applications, trace dynamic behavior first so reflection/resources are registered:

```bash
java -agentlib:native-image-agent=config-output-dir=META-INF/native-image \
     -jar app.jar              # exercise the app, then:
native-image -jar app.jar
```

## Architecture / How It Works

Three subsystems dominate the repo:

1. **Graal compiler** (`compiler/`) — a JIT compiler that plugs into HotSpot through the **JVMCI** interface (JDK-standard since JDK 9). It consumes bytecode, builds a sea-of-nodes IR ("Graal IR"), optimizes, and emits machine code. It can run itself as Java bytecode (compiled by C2 first, "jargraal") or as a Native Image of the compiler ("libgraal"), which avoids compiler warmup and its own GC interference.

2. **Native Image / SubstrateVM** (`substratevm/`) — an AOT pipeline built on a **closed-world assumption**: it performs a static points-to analysis over the whole program from `main`, compiles every reachable method ahead of time, and snapshots the initialized heap into the binary. Everything the analysis cannot see — reflection, JNI, dynamic proxies, resources, serialization — must be declared via reachability metadata or it will not exist at runtime. The produced binary bundles SubstrateVM: its own GC, thread scheduling, and runtime, not HotSpot.

3. **Truffle** (`truffle/`) — an AST-interpreter framework. Interpreters written against it are partially evaluated by the Graal compiler at run time; hot ASTs become optimized machine code. `TRegex` (`regex/`), `Sulong` (`sulong/`, LLVM bitcode), `GraalWasm` (`wasm/`), and `Espresso` (`espresso/`, a meta-circular Java interpreter) all sit here. The Polyglot API lets these languages share one heap and call each other.

The tight coupling is deliberate: the same Graal IR and backend are the substrate for JIT, AOT, and every guest language. A change to the compiler can move throughput, native binary size, and interpreter performance at once, which is why the project gates changes through an extensive internal CI ("Gate").

## Production Notes

**Native Image is a different runtime, not a faster JVM.** The closed-world model breaks any code that discovers classes at run time. Frameworks that were not designed for it (heavy reflection, runtime bytecode generation, classpath scanning) either need agent-generated metadata or simply do not work. The ecosystem answer is framework support — Spring Boot 3+, Quarkus, Micronaut, and Helidon generate the metadata at build time; the reachability-metadata repository ships configs for common libraries. Outside those, expect to debug `ClassNotFoundException`/`missing reflection registration` at run time.

**Peak throughput can be lower than the JIT.** A JIT profiles the live workload and re-optimizes; AOT compiles once with no runtime profile. For latency-sensitive services this is a win (no warmup, flat curve); for long-running throughput-bound workloads a warmed-up HotSpot with C2/Graal JIT often wins unless you apply PGO — and full PGO plus G1 GC for Native Image are **Oracle GraalVM features, not in this repo**. Community Edition native images default to a Serial GC.

**Build cost is real.** `native-image` builds are slow and memory-hungry: large applications routinely need several GB of RAM and minutes of build time per binary, and the build must be run for each target OS/architecture (no cross-compilation). This reshapes CI pipelines and Docker image builds.

**Build-time vs. run-time initialization** is the subtle footgun. Class initializers that run at build time capture state into the image heap; anything that reads environment, opens files, or seeds randomness at `<clinit>` can bake stale or host-specific values into the binary. Getting the `--initialize-at-build-time` / `--initialize-at-run-time` split right is a recurring source of heisenbugs.

**Versioning changed midstream.** Releases were `19.x`–`22.x` (calendar-ish) through 2022, then switched to a JDK-aligned scheme — "GraalVM for JDK 17/21/24" — tracking the OpenJDK release train[^3]. Old `gu install` component management and the `-Enterprise` naming are gone in the new line, which breaks scripts and docs written against the pre-2023 layout.

## When to Use / When Not

**Use when:**
- You need fast startup and low, flat memory for short-lived or scale-to-zero workloads (CLIs, serverless, microservices) — Native Image's core payoff.
- You are on a framework (Quarkus, Micronaut, Spring Boot native) that already handles the reachability metadata for you.
- You want the Graal JIT's throughput on a normal JVM without going native.
- You are embedding or building a language and want Truffle's compiler for free.

**Avoid when:**
- Your app relies on heavy runtime reflection, dynamic class loading, or agents that Native Image's closed world cannot see, and it is not on a supporting framework.
- You are throughput-bound and long-running: a warmed JIT may beat a CE native image without PGO.
- Your CI cannot absorb multi-GB, multi-minute per-target native builds.
- You need the advanced Native Image performance features (G1, full PGO) but cannot adopt the Oracle GraalVM license.

## Alternatives

- openjdk/jdk — the mainstream JVM; use it when you do not need AOT and want the most compatible, best-understood runtime.
- oracle/graalpython, oracle/graaljs — the guest languages built on this repo, when you want polyglot embedding rather than the JDK itself.
- Project Leyden (in OpenJDK) — use when you want AOT/startup gains inside the standard JDK without leaving the closed-world-free model.
- quarkus/quarkus / micronaut-projects/micronaut-core — pick these when your real goal is native microservices; they wrap Native Image with the metadata plumbing done for you.
- ziglang/zig or rust-lang/rust — use instead when you want a natively-compiled language from the start rather than retrofitting AOT onto the JVM.

## History

| Version | Date | Notes |
|---------|------|-------|
| research | 2011–2018 | Graal compiler + Truffle developed at Oracle Labs; shipped experimentally in the JDK via JVMCI. |
| 19.0 | 2019-05 | First unified GraalVM release (CE and Enterprise), Native Image and polyglot as products[^1]. |
| 20.x | 2020 | Native Image maturing; broader language support. |
| 21.x | 2021 | Truffle language versions, tooling improvements. |
| 22.3 | 2022-11 | Last release under the old calendar-style versioning line. |
| for JDK 17 | 2023-06 | Switch to JDK-aligned versioning; "Oracle GraalVM" replaces "Enterprise Edition"[^2][^3]. |
| for JDK 21 | 2023-09 | Aligned with the OpenJDK 21 LTS train. |
| for JDK 24/25 | 2025 | Continued JDK-cadence releases; ongoing Native Image and compiler work. |

## References

[^1]: GraalVM project overview and history. https://www.graalvm.org/
[^2]: "GraalVM Editions" — Community Edition vs. Oracle GraalVM licensing and feature split. https://www.graalvm.org/community/
[^3]: "New Release Model for GraalVM" (JDK-aligned versioning, 2023). https://medium.com/graalvm/a-new-graalvm-release-and-new-free-license-4aab483692f5

## Tags

java, jvm, compiler, aot, jit, native-image, graalvm, truffle, polyglot, oracle-labs, gpl
