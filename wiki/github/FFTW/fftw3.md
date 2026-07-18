# FFTW/fftw3

> The "Fastest Fourier Transform in the West" — the C library that made adaptive, self-optimizing FFTs the default expectation for numerical code.

[GitHub repo](https://github.com/FFTW/fftw3) ·
[Official website](https://www.fftw.org) ·
[License: GPL-2.0-or-later](https://github.com/FFTW/fftw3/blob/master/COPYING) (commercial licenses available via MIT)

## Overview

FFTW computes discrete Fourier transforms in O(N log N) for arbitrary input sizes — including primes — in one or more dimensions, for real and complex data, in single, double, and long-double precision. It was written at MIT by Matteo Frigo and Steven G. Johnson, first released in 1997, and won the 1999 J. H. Wilkinson Prize for Numerical Software[^1]. Version 3.x (2003) is a ground-up rewrite whose design is documented in a Proceedings of the IEEE paper that remains the canonical reference[^2]. FFTW is the de facto FFT backend of scientific computing: MATLAB ships it, Julia wraps it (FFTW.jl), and countless simulation, imaging, and signal-processing codes link it.

The defining idea is that no single FFT implementation is fastest everywhere. Instead of one hard-coded algorithm, FFTW ships a *planner* that, at runtime, searches over many decompositions of your specific transform on your specific hardware and composes a "plan" from a library of pre-generated code fragments. You pay a one-time planning cost (microseconds to minutes, your choice) in exchange for execution speed competitive with vendor-tuned libraries[^2].

The GitHub repository carries an unusual warning: do not build from it. Most of FFTW's C source is machine-generated, and the repo contains only the *generator*; users are expected to download official tarballs from fftw.org, which contain the generated code and build with a plain C compiler[^3]. Stars (~3.1k) understate usage — most users never touch the repo. Maintenance is slow but real: commits continue into 2026, though the last tagged release (3.3.10) dates to September 2021, and 181 issues sit open.

## Getting Started

```bash
# Package managers (recommended over building from the git repo)
sudo apt install libfftw3-dev     # Debian/Ubuntu
brew install fftw                 # macOS

# Or from the official tarball (NOT the git repo)
./configure --enable-shared && make && sudo make install
```

```c
#include <fftw3.h>

int main(void) {
    int n = 1024;
    fftw_complex *in  = fftw_malloc(sizeof(fftw_complex) * n);
    fftw_complex *out = fftw_malloc(sizeof(fftw_complex) * n);

    /* Plan FIRST, then fill input: FFTW_MEASURE overwrites arrays. */
    fftw_plan p = fftw_plan_dft_1d(n, in, out, FFTW_FORWARD, FFTW_MEASURE);

    /* ... fill in[] ... */
    fftw_execute(p);              /* out[] now holds the unnormalized DFT */

    fftw_destroy_plan(p);
    fftw_free(in); fftw_free(out);
    return 0;
}
```

Link with `-lfftw3 -lm` (`-lfftw3f` for single precision, `-lfftw3_threads` or `-lfftw3_omp` for the threaded transforms).

## Architecture / How It Works

FFTW is two programs. **genfft**, written in OCaml, is a special-purpose compiler that generates "codelets" — straight-line, unrolled C functions computing small fixed-size transforms, with algebraic simplification and scheduling done symbolically at generation time[^2]. The distributed tarballs contain the generated C; the git repo contains genfft itself, which is why a naive `configure; make` from GitHub fails[^3].

The **planner** is the runtime half. Given a transform (size, rank, stride, direction, precision), it recursively decomposes it via Cooley-Tukey and other identities into a tree of codelet applications, then selects among the possible trees. Rigor levels trade planning time for plan quality: `FFTW_ESTIMATE` uses heuristics; `FFTW_MEASURE` times candidate plans on your actual hardware; `FFTW_PATIENT` and `FFTW_EXHAUSTIVE` widen the search. Measured plans can be serialized as **wisdom** and reloaded, amortizing planning across runs. Prime and near-prime sizes are handled by Rader's and Bluestein's algorithms, so worst-case complexity stays O(N log N).

Execution is decoupled from planning: `fftw_execute` is the hot path and is thread-safe. SIMD (SSE2/AVX/AVX2/AVX-512, NEON, VSX) lives inside the codelets and is selected at configure and plan time. Parallelism comes in layers — shared-memory threads (pthreads or OpenMP) and distributed-memory MPI transforms (since 3.3) — each as a separate library. The API surface is wide but mechanical: every combination of precision (`fftwf_`/`fftw_`/`fftwl_`), domain (c2c, r2c/c2r, r2r including DCT/DST), and rank has its own planner entry point.

## Production Notes

**Planning is the API's sharp edge.** `FFTW_MEASURE` destroys the contents of input/output arrays during planning — plan before filling data, or use `FFTW_ESTIMATE` (slower plans, no clobber). Plans are bound to the pointer alignment and strides they were created with; executing a plan on different arrays via `fftw_execute_dft` is legal only if alignment matches (use `fftw_malloc`/`fftw_alignment_of` to be safe).

**The planner is not thread-safe.** `fftw_execute` may be called from many threads, but plan creation/destruction must be serialized by the caller or via `fftw_make_planner_thread_safe()`. A recurring source of heisenbugs in threaded applications.

**Transforms are unnormalized.** A forward-then-backward round trip scales the data by N. Every FFTW user rediscovers this once.

**Multi-dimensional c2r transforms destroy their input** by default; `FFTW_PRESERVE_INPUT` avoids this but is not available for all rank/type combinations — otherwise keep a copy.

**Wisdom is machine-specific.** Wisdom exported on one microarchitecture misleads the planner on another; shipping wisdom with your application is only safe for homogeneous fleets. System wisdom via the `fftw-wisdom` tool helps on managed clusters.

**Licensing is the other footgun.** FFTW is GPL-2.0-or-later — linking it into proprietary software requires purchasing a commercial license from MIT or picking a different library[^4]. This is why NumPy/SciPy use pocketfft rather than FFTW, and why Intel MKL provides an FFTW-compatible wrapper interface.

**Release cadence is glacial.** 3.3.10 (2021) is current; expect distro packages to be effectively identical for years. The upside is a very stable API/ABI; the downside is that support for new ISAs arrives slowly.

## When to Use / When Not

**Use when:**
- You need best-in-class CPU FFT performance across arbitrary sizes, strides, and dimensions, and can amortize planning cost over many executions.
- You are on GPL-compatible terms (open-source scientific code, internal tools) or can buy the MIT commercial license.
- You need real-input, DCT/DST, or MPI-distributed transforms behind one API.

**Avoid when:**
- You ship proprietary binaries and cannot license commercially — use pocketfft, KISS FFT, or a vendor library instead.
- Transforms run on GPU — cuFFT/rocFFT/VkFFT are the right tools.
- You do a handful of one-off transforms where planning overhead dominates, or you need a tiny embeddable dependency.

## Alternatives

- mreineck/pocketfft — BSD-licensed C++/C; NumPy and SciPy's backend; use when GPL is a blocker and near-FFTW speed suffices.
- mborgerding/kissfft — small, simple, BSD-licensed; use when you want a library you can vendor and audit rather than maximum speed.
- DTolm/VkFFT — Vulkan/CUDA/HIP/OpenCL GPU FFTs; use when the data already lives on a GPU.
- ejmahler/RustFFT — pure-Rust, safe, SIMD-accelerated; use in Rust projects that want no C dependency.
- Intel oneMKL FFT — proprietary but freely redistributable, offers an FFTW3-compatible interface; use on x86 when licensing rules out FFTW.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 1997 | Initial release at MIT (Frigo & Johnson)[^1]. |
| 2.0 | 1998 | Second-generation API; basis of the 1999 Wilkinson Prize win. |
| 3.0 | 2003-04 | Complete rewrite: new incompatible API, SIMD codelets, r2r transforms[^2]. |
| 3.3 | 2011-07 | MPI distributed-memory transforms, AVX support. |
| 3.3.8 | 2018-05 | Maintenance release; broadened SIMD coverage. |
| 3.3.10 | 2021-09 | Latest release as of 2026; bug fixes[^5]. |

## References

[^1]: FFTW home page and project history. https://www.fftw.org/
[^2]: Matteo Frigo and Steven G. Johnson, "The Design and Implementation of FFTW3", Proceedings of the IEEE 93(2), 2005. https://www.fftw.org/fftw-paper-ieee.pdf
[^3]: FFTW/fftw3 README — "DO NOT CHECK OUT THESE FILES FROM GITHUB UNLESS YOU KNOW WHAT YOU ARE DOING." https://github.com/FFTW/fftw3#readme
[^4]: FFTW FAQ — licensing and commercial use. https://www.fftw.org/faq/section1.html
[^5]: FFTW release notes. https://www.fftw.org/release-notes.html

## Tags

c, fft, fourier-transform, signal-processing, numerical-computing, hpc, scientific-computing, simd, mpi, code-generation, gpl
