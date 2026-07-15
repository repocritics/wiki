# apache/commons-math

> A broad general-purpose mathematics and statistics library for Java — comprehensive, widely depended-on, and effectively frozen at a 2016 release while its successor stalls.

[GitHub repo](https://github.com/apache/commons-math) ·
[Official website](https://commons.apache.org/math) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

Commons Math is the Apache Commons component for self-contained mathematics and
statistics that the Java standard library does not cover: descriptive and
inferential statistics, linear algebra, numerical optimization, ODE solvers,
interpolation and curve fitting, probability distributions, random data
generation, clustering, computational geometry, complex numbers, special
functions, and FFT/transforms. It has no third-party runtime dependencies — a
deliberate design goal that made it an easy transitive dependency across a large
share of the Java ecosystem[^1].

The defining tension is maturity versus abandonment. The last official release,
**3.6.1, shipped in March 2016** and the project itself states it is "quite old
and not supported anymore"[^2]. The intended successor, 4.0, has been under
development for many years without a final release, and its low-level pieces have
been split off into separate components — Commons Numbers, Commons RNG, Commons
Geometry, and Commons Statistics — each with bug fixes and API improvements not
back-ported to 3.x[^2]. In practice most users still pull `commons-math3`, a
frozen artifact, while active development is fragmented across the successor
components and an out-of-tree fork.

Commons Math is best understood as a stable, encyclopedic toolbox for
non-performance-critical numerical work, not as a living library you should
expect fixes or new features from.

## Getting Started

```xml
<!-- Maven — the last stable release -->
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-math3</artifactId>
  <version>3.6.1</version>
</dependency>
```

```java
import org.apache.commons.math3.stat.descriptive.DescriptiveStatistics;
import org.apache.commons.math3.stat.regression.SimpleRegression;

DescriptiveStatistics stats = new DescriptiveStatistics();
for (double v : new double[]{2, 4, 4, 4, 5, 5, 7, 9}) stats.addValue(v);
double mean = stats.getMean();               // 5.0
double sd   = stats.getStandardDeviation();  // sample sd
double p95  = stats.getPercentile(95);

SimpleRegression reg = new SimpleRegression();
reg.addData(1, 2); reg.addData(2, 3); reg.addData(3, 5);
double slope = reg.getSlope();               // least-squares fit
```

Note the `math3` in the package name: the 3.x line lives entirely under
`org.apache.commons.math3.*`, and 4.x uses `org.apache.commons.math4.*`. This
version-in-package convention is intentional so incompatible majors can coexist
on one classpath.

## Architecture / How It Works

Commons Math is a flat collection of loosely coupled sub-packages rather than a
framework. The main areas are `stat` (descriptive/inference/regression),
`linear` (real and field matrices, decompositions), `optim` / `fitting`
(optimizers, least squares, curve fitting), `ode` (integrators),
`analysis` (functions, interpolation, integration, differentiation),
`distribution`, `random`, `geometry`, `ml` (clustering), `complex`, `special`,
and `transform`. Most sub-packages can be used in isolation.

Two design decisions dominate the codebase. First, **zero runtime
dependencies** — everything is implemented in-tree, which is why it is so widely
vendored but also why algorithm quality varies package to package. Second,
heavy use of `double`-based dense representations: the linear algebra is
textbook-correct (LU, QR, Cholesky, eigen, SVD) but not tuned for large or
sparse matrices, and there is no native BLAS backing.

The 4.0 effort is a decomposition, not a rewrite. Random number generation moved
to Commons RNG, primitive numeric types and special functions to Commons
Numbers, geometry to Commons Geometry, and statistics to Commons Statistics[^2].
What remains in Commons Math proper is meant to be modularized on top of those.
Because 4.0 never shipped a final, this leaves the ecosystem in a split state:
the stable code you can depend on (3.6.1) predates the reorganization, and the
reorganized code lives in components at different maturity levels.

## Production Notes

**You are almost certainly using an unmaintained artifact.** `commons-math3`
3.6.1 receives no fixes. Known numerical edge cases and the occasional
correctness bug filed in the MATH JIRA will not be patched in a 3.x release.
Treat it as a stable-but-final dependency and pin the version.

**Performance is adequate, not competitive.** The linear algebra and
optimization routines are correct reference implementations but are
single-threaded, dense, pure-Java, and not SIMD/BLAS-accelerated. For large
matrix workloads, sparse systems, or hot numerical loops, EJML, ojAlgo, or a
native-backed stack (nd4j/BLAS) are substantially faster. Commons Math is fine
for modest problem sizes and glue code, not for a numerical core.

**Watch the package coordinates.** Old code and old answers reference
`commons-math` (1.x/2.x, package `org.apache.commons.math`) versus
`commons-math3` (`org.apache.commons.math3`). These are different artifacts with
different group/artifact IDs and are not drop-in compatible. Mixing them, or
copy-pasting a 2.x snippet into a 3.x project, produces confusing import errors.

**Migrating off 3.x is not a version bump.** Moving to the successor components
means depending on Commons Numbers / RNG / Geometry / Statistics with new package
names and reorganized APIs, or switching to the Hipparchus fork. There is no
in-place upgrade path from `commons-math3` to a supported release.

**Random number generation is the clearest thing to move.** If you only use
`RandomDataGenerator` / `RandomGenerator`, Commons RNG is the maintained,
faster, better-documented replacement and is worth adopting on its own.

## When to Use / When Not

**Use when:**
- You need a dependency-free grab bag of standard numerical/statistical routines
  for modest data sizes.
- You want stable, unchanging behavior and are comfortable pinning a final
  release.
- Your use is glue-level: summary stats, a regression, an interpolation, a
  distribution CDF, a small solver.

**Avoid when:**
- You need ongoing maintenance, security fixes, or new features.
- Performance matters — large/sparse linear algebra, heavy optimization, or
  tight numerical loops.
- You are starting fresh and can adopt the maintained successor components
  (Numbers/RNG/Geometry/Statistics) or a purpose-built library instead.

## Alternatives

- Hipparchus-Math/hipparchus — use instead when you want the actively maintained
  continuation of Commons Math 3's ODE, optimization, and geometry code (it began
  as a fork of Commons Math).
- lessthanoptimal/ejml — use instead when linear algebra performance matters;
  dense and sparse, tuned for speed.
- optimatika/ojAlgo — use instead for fast linear algebra plus optimization
  (LP/QP/MIP) in pure Java.
- deeplearning4j/nd4j — use instead when you want native BLAS-backed
  n-dimensional arrays for large numerical workloads.
- haifengl/smile — use instead when your real goal is machine learning and stats
  rather than low-level numerical primitives.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2004-12 | First release under Apache Commons; package `org.apache.commons.math`. |
| 2.0 | 2009-08 | Large API expansion (optimization, ODE, geometry). |
| 3.0 | 2012-03 | Package moved to `org.apache.commons.math3` for coexistence. |
| 3.5 | 2015-11 | Late 3.x maintenance release. |
| 3.6 | 2016-03 | Final feature release of the 3.x line. |
| 3.6.1 | 2016-03 | Last official release; now unsupported[^2]. |
| 4.0 | unreleased | In development for years; low-level code split into Numbers/RNG/Geometry/Statistics[^2]. |

## References

[^1]: Apache Commons Math homepage and user guide. https://commons.apache.org/proper/commons-math/userguide/index.html
[^2]: Apache Commons Math README — status of 3.6.1 and the 4.0 component split. https://github.com/apache/commons-math/blob/master/README.md

## Tags

java, mathematics, statistics, linear-algebra, optimization, numerical-computing, apache-commons, curve-fitting, ode-solver, unmaintained, jvm
