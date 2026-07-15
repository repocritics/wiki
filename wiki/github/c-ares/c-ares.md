# c-ares/c-ares

> An asynchronous C DNS stub-resolver library — the name-resolution layer inside Node.js, curl, and much of the networked C ecosystem.

[GitHub repo](https://github.com/c-ares/c-ares) ·
[Official website](https://c-ares.org/) ·
[License: MIT](https://github.com/c-ares/c-ares/blob/main/LICENSE.md)

## Overview

c-ares is a small C library that performs DNS queries without blocking the
calling thread. It is a *stub* resolver: it formats queries, talks to whatever
recursive resolvers the system is configured to use (`/etc/resolv.conf`, the
Windows registry, macOS system config), parses the responses, and hands results
back through callbacks. It does not do recursion or DNSSEC validation itself.

The library exists because the POSIX `getaddrinfo()`/`gethostbyname()` calls are
synchronous and block the thread until a resolver replies — unacceptable for an
event-loop program that needs to resolve many names concurrently. c-ares began
as `ares` (Greg Hudson, MIT), which Daniel Stenberg forked around 2004 as
"c-ares" to keep it maintained for curl; the GitHub repository's 2010 creation
date reflects a later migration, not the project's true age, which exceeds 20
years[^1]. Brad House is the current lead maintainer and drove a substantial
internal modernization starting with the 1.22 series[^2].

The defining tension is that c-ares gives you correctness and control but no
convenience: it is a *library*, not a service. You own the event loop, the
socket registration, and the timeout bookkeeping. In exchange you get a resolver
that behaves identically across Linux, the BSDs, macOS, Solaris, AIX, Windows,
Android, and iOS, builds with any C89 compiler, and is continuously fuzzed via
OSS-Fuzz[^3]. That portability is why it is embedded rather than reimplemented —
Node.js uses it for its `dns.resolve*()` family, and curl uses it for
asynchronous name resolution when built with `--enable-ares`.

## Getting Started

```bash
# Distro packages
apt-get install libc-ares-dev        # Debian/Ubuntu
brew install c-ares                  # macOS

# From source (CMake is the supported modern path)
cmake -B build && cmake --build build && cmake --install build
```

```c
#include <ares.h>
#include <stdio.h>
#include <string.h>

static void cb(void *arg, int status, int timeouts,
               struct ares_addrinfo *result) {
    if (status != ARES_SUCCESS) {
        fprintf(stderr, "lookup failed: %s\n", ares_strerror(status));
        return;
    }
    for (struct ares_addrinfo_node *n = result->nodes; n; n = n->ai_next)
        printf("family=%d addrlen=%d\n", n->ai_family, (int)n->ai_addrlen);
    ares_freeaddrinfo(result);
}

int main(void) {
    ares_channel_t *channel;
    struct ares_options opts;
    memset(&opts, 0, sizeof(opts));

    ares_library_init(ARES_LIB_INIT_ALL);
    opts.evsys = ARES_EVSYS_DEFAULT;                 // built-in event thread
    ares_init_options(&channel, &opts, ARES_OPT_EVENT_THREAD);

    struct ares_addrinfo_hints hints = {0};
    hints.ai_family = AF_UNSPEC;
    ares_getaddrinfo(channel, "example.com", NULL, &hints, cb, NULL);

    ares_queue_wait_empty(channel, -1);              // block until drained
    ares_destroy(channel);
    ares_library_cleanup();
    return 0;
}
```

## Architecture / How It Works

The central object is a **channel** (`ares_channel_t`), created with
`ares_init_options()`. It holds configuration (server list, timeouts, search
domains, sortlist) and tracks in-flight queries. Each query is a callback
registration — you call `ares_getaddrinfo()`, `ares_query()`, `ares_search()`,
etc., and c-ares invokes your callback later, from inside whatever code advances
the channel.

Historically that advancement was your job, and c-ares offers three integration
generations that still coexist:

1. **`ares_fd()` / `ares_process()`** — the original select-based model, capped
   by `FD_SETSIZE`. Legacy and poll-heavy.
2. **`ares_getsock()` / `ares_process_fd()`** — hands you up to
   `ARES_GETSOCK_MAXNUM` (16) socket fds with their read/write interest to
   register in epoll/kqueue, plus `ares_timeout()` for the next deadline. This is
   what most embedders (including Node's libuv integration) wire up.
3. **Event-thread mode (`ARES_OPT_EVENT_THREAD`)** — recent releases let c-ares
   spawn its own internal event thread (epoll/kqueue/IOCP), so an application
   with no event loop can still resolve asynchronously; `ARES_EVSYS_*` selects
   the backend[^2].

Since the 1.22 rewrite, DNS messages flow through a unified record abstraction
(`ares_dns_record_t`) with dedicated safe parsers and writers, replacing the old
per-record-type buffer code that was a recurring source of memory bugs. The RFC
surface is broad — A/AAAA, SRV, NAPTR, CAA, TLSA, SVCB/HTTPS (RFC 9460), DNS
cookies (RFC 7873), 0x20 case randomization, and DNSSEC record *parsing* — but
it parses DNSSEC RRs without performing validation[^4].

## Production Notes

**You must drive it, or let it drive itself.** The single most common
integration mistake is forgetting `ares_timeout()`/`ares_process_fd()` and
wondering why queries never complete or never retry. If you don't want an event
loop at all, use the event-thread option; if you have one, use `ares_getsock()`
and never the legacy `ares_fd()` path.

**Thread-safety is opt-in and recent.** For most of its life a channel was not
safe to use from multiple threads; each thread needed its own channel. Recent
1.2x releases added a thread-safety build/runtime option — verify it is enabled
in your build before sharing a channel[^2]. The 16-socket `ares_getsock()` limit
also surprises programs that fan out many parallel queries through a single
channel; queries queue behind available sockets rather than failing.

**Security history is real but the trend is good.** c-ares has shipped several
CVEs over the years — sortlist configuration buffer issues, a 0-length UDP DoS,
weak PRNG seeding, and config-file parsing bugs among them. The modern parser
rewrite plus continuous OSS-Fuzz coverage and CII Best Practices compliance have
hardened it considerably, but it is C parsing untrusted network input: keep it
patched.

**You rarely upgrade it directly.** Because c-ares is vendored inside Node.js,
curl, and distro packages, the version you run is usually chosen by that host —
a c-ares CVE becomes a Node/curl update on your schedule, not an independent
`apt upgrade`. Check `ares_version()` at runtime rather than trusting the system
package version.

## When to Use / When Not

**Use when:**
- You are writing C/C++ networked code and need non-blocking name resolution
  across many platforms with one API.
- You need control over servers, timeouts, search domains, or record types that
  `getaddrinfo()` doesn't expose.
- You are already inside an event loop and want DNS to participate in it.

**Avoid when:**
- You need DNSSEC *validation* or a full recursive/caching resolver — c-ares is
  a stub and validates nothing.
- Blocking resolution on a threadpool is acceptable (a plain `getaddrinfo()` in
  a worker thread is far less code).
- You want DNS-over-TLS/HTTPS as a first-class transport — that is not c-ares's
  design center.

## Alternatives

- NLnetLabs/unbound — full validating, recursive, caching resolver; use instead when you need DNSSEC validation and a real cache, not just stub queries.
- getdnsapi/getdns — modern async resolver API (built atop unbound) with DNSSEC and DNS-over-TLS; use when you want validation plus a convenient async interface.
- NLnetLabs/ldns — DNS record parsing/building and resolver toolkit; use when you are constructing DNS tooling rather than resolving names in a client.
- libuv/libuv — its threadpool `uv_getaddrinfo()` wraps system `getaddrinfo()`; use when a blocking syscall on a worker thread is enough and you don't want a DNS dependency.
- GNU adns — older asynchronous stub resolver; niche, mostly of historical interest versus c-ares's active maintenance.

## History

| Version | Date | Notes |
|---------|------|-------|
| ares | c. 1998 | Predecessor by Greg Hudson (MIT). |
| c-ares fork | c. 2004 | Daniel Stenberg forks and renames for curl[^1]. |
| 1.22.x | 2023 | Start of Brad House internal rewrite: unified record parser/writer[^2]. |
| 1.29.0 | 2024-05-24 | Release signed per README example[^5]. |
| 1.34.x | 2024–2025 | Current line; SLSA provenance on releases, event-thread + thread-safety options mature. |

## References

[^1]: c-ares overview and history, c-ares.org. https://c-ares.org/
[^2]: c-ares GitHub repository and release notes (maintainer Brad House). https://github.com/c-ares/c-ares/releases
[^3]: OSS-Fuzz continuous fuzzing of c-ares. https://github.com/google/oss-fuzz
[^4]: c-ares README, "Supported RFCs and Proposals" (DNSSEC parsing only, no validation). https://github.com/c-ares/c-ares/blob/main/README.md
[^5]: c-ares README release-signature example referencing c-ares-1.29.0 (2024-05-24). https://github.com/c-ares/c-ares/blob/main/README.md

## Tags

c, dns, resolver, async, networking, stub-resolver, library, cross-platform, node-js, curl, event-loop
