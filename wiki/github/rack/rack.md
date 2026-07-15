# rack/rack

> A modular Ruby web server interface — the `[status, headers, body]` contract every Ruby web framework sits on.

[GitHub repo](https://github.com/rack/rack) ·
[Documentation](https://rack.github.io/rack/) ·
[License: MIT](https://github.com/rack/rack/blob/main/MIT-LICENSE)

## Overview

Rack is the interface layer between Ruby web servers and Ruby web applications. It was created by Christian (now Leah) Neukirchen in 2007, modeled directly on Python's WSGI (PEP 333)[^1]. Its entire premise fits in one sentence: a Rack application is any object that responds to `call(env)` and returns a three-element array `[status, headers, body]`. Everything else — routing, middleware, request/response wrappers — is convention layered on top of that call.

Almost the entire Ruby web ecosystem depends on Rack. Rails, Sinatra, Hanami, Roda, Padrino, and Camping are all Rack applications; Puma, Falcon, Unicorn, Pitchfork, Passenger, and Thin are all Rack servers. Because it is the shared contract, a valid Rack app runs unchanged across every compliant server. This near-universal position is why Rack is boring by design: it changes slowly, guards backward compatibility fiercely, and treats a breaking change as a multi-year migration event rather than a release note.

The defining tension is scope discipline. Rack is deliberately not a framework — it ships a request/response wrapper, a middleware stack builder, and a set of small middleware (logging, ETag, static files, deflate), and stops there. Session handling, the `rackup` runner, and much of the historical grab-bag were extracted into separate gems in Rack 3 to keep the core minimal[^2]. That minimalism is the appeal and the friction: you get a stable substrate, but anything resembling convenience lives one dependency away.

## Getting Started

```bash
gem install rack
# session handling and the CLI runner are separate gems now:
gem install rack-session rackup
```

Create `config.ru`:

```ruby
# config.ru — a minimal Rack app is anything responding to #call
run do |env|
  request = Rack::Request.new(env)
  [200, { "content-type" => "text/plain" }, ["Hello #{request.path}"]]
end
```

```bash
rackup          # boots the app on http://localhost:9292
curl localhost:9292/world   # => Hello /world
```

A middleware is just an app that wraps another app:

```ruby
class RequestTimer
  def initialize(app) = @app = app

  def call(env)
    started = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    status, headers, body = @app.call(env)
    headers["x-runtime"] = (Process.clock_gettime(Process::CLOCK_MONOTONIC) - started).to_s
    [status, headers, body]
  end
end

use RequestTimer
run MyApp
```

## Architecture / How It Works

The whole system is three ideas:

1. **The app contract.** An app responds to `call(env)` where `env` is a Hash of CGI-style keys (`REQUEST_METHOD`, `PATH_INFO`, `rack.input`, …) and returns `[status, headers, body]`. `body` responds to `each` (buffered) or, since Rack 3, to `call(stream)` (streaming). That's the entire spec surface applications must honor[^3].

2. **Middleware as onion layers.** Each middleware takes the next app in its constructor and returns a modified `[status, headers, body]`. `Rack::Builder` (driven by `config.ru`'s `use`/`run` DSL) composes them into a single callable. There is no registry, no lifecycle, no dependency injection — just nested `call`.

3. **Convenience wrappers.** `Rack::Request` and `Rack::Response` wrap the raw `env` and the response triple with query parsing, cookie handling, and multipart decoding. `Rack::MockRequest`/`MockResponse` let you exercise an app in-process with no socket.

`Rack::Lint` is the enforcement mechanism: wrap any app in it during development and it raises on spec violations (wrong header casing, non-rewindable input, malformed status). Framework and server authors rely on it as the conformance test. `Rack::URLMap` and `Rack::Cascade` provide primitive dispatch — mounting apps by path prefix and falling through on 404 respectively — which is how frameworks mount sub-apps.

Rack does no I/O itself. It never opens a socket; the server (Puma, Falcon, …) parses HTTP and constructs `env`, and Rack is the shape of what gets passed. This is why Rack is both ubiquitous and nearly invisible in production stacks.

## Production Notes

**The Rack 3 header casing break.** Rack 3 requires response header keys to be lowercase strings (`content-type`, not `Content-Type`) and requires the headers object to be a plain mutable Hash[^2]. Middleware that hard-coded capitalized header names silently stopped matching. This is the single most common Rack 3 upgrade failure and it fails quietly — a `Set-Cookie` written as `"Set-Cookie"` simply doesn't collide with framework-written `"set-cookie"`, producing duplicate or dropped headers rather than a crash.

**Extracted gems.** Upgrading to Rack 3 breaks any app that referenced `Rack::Session`, `Rack::Server`, or the `rackup` binary from the `rack` gem directly. These moved to `rack-session` and `rackup`. `Rack::File` was renamed `Rack::Files`; `Rack::Server` became `Rackup::Server`. Bundlers won't warn — you get a `NameError` at boot.

**Ruby 3.4 `base64` warning.** On Rack 2.2.x under Ruby 3.4+, `base64` is no longer a default gem and Rack emits a load warning or error. Add `base64` to your Gemfile explicitly, or upgrade to Rack 3.1+[^4].

**Parameter-parsing DoS limits.** Rack ships hard limits to bound resource exhaustion from crafted requests: `param_depth_limit` (nesting, default 32), `multipart_file_limit` (default 128 files), `multipart_total_part_limit` (default 4096 parts), and the `RACK_QUERY_PARSER_*` bytesize/params limits. These exist because query and multipart parsing have been a recurring CVE surface for Rack; several past advisories were parser-resource issues. Raising these limits to accommodate large forms reopens that surface — tune deliberately, not reflexively.

**Version support is narrow.** Only 3.2.x receives full bug fixes; 3.1.x and 2.2.x get security patches only; 3.0.x and everything ≤2.1 are end-of-life[^4]. Because Rack sits under Rails, a Rack CVE effectively means "patch every Ruby web app you run," so staying on a supported line matters more than the low release cadence suggests.

## When to Use / When Not

**Use when:**
- You're writing a Ruby web framework, a server adapter, or middleware — Rack is the mandatory shared contract.
- You want a tiny HTTP app or webhook receiver with no framework overhead: a `config.ru` with a lambda is a complete service.
- You need to insert cross-cutting behavior (auth, logging, rate limiting) beneath an existing Rails/Sinatra app.

**Avoid when:**
- You want routing, ORM, views, or any application structure — Rack is intentionally none of these; reach for Sinatra, Roda, or Rails.
- You're not in Ruby: the contract is Ruby-specific (the concept maps to WSGI in Python, but the code does not port).
- You need HTTP/2 or async streaming as first-class primitives — that lives in the server (Falcon) and the Rack 3 streaming body, not in Rack's traditional buffered model.

## Alternatives

- rack/rackup — the extracted CLI runner and `Rackup::Server`; pair with Rack rather than replace it when you need the `rackup` command.
- rack/rack-contrib — community middleware collection; use when you need common middleware (JSON body parsing, locale, etc.) that core Rack deliberately omits.
- socketry/falcon — a Rack-compatible server built on async fibers; use instead of Puma when you need high-concurrency streaming or HTTP/2.
- roda / sinatra — use one of these instead of raw Rack when you want routing and a request DSL but still want to stay close to the metal.
- pallets/werkzeug — the Python WSGI equivalent; the reference point if you're comparing across ecosystems rather than staying in Ruby.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2007-03 | First release; WSGI-inspired `call(env)` contract[^1]. |
| 1.0 | 2009-04 | First stable line; adopted broadly across Ruby frameworks. |
| 1.6 | 2015-03 | Last of the 1.x line before the 2.0 reset. |
| 2.0 | 2016-05 | Frozen-string headers, streaming refinements, Ruby 2.2+ baseline. |
| 3.0 | 2022-09 | Lowercase header keys, Hash headers, streaming bodies, `rackup`/`rack-session` extracted[^2]. |
| 3.1 | 2024-02 | Continued 3.x hardening and parser limits. |
| 3.2 | latest | Bug fixes and security patches; current fully-supported line[^4]. |

## References

[^1]: Rack originated from Christian Neukirchen's 2007 work adapting Python's WSGI (PEP 333) idea to Ruby. https://rack.github.io/rack/
[^2]: Rack 3 Upgrade Guide — header casing, Hash headers, streaming body protocol, and gem extraction. https://github.com/rack/rack/blob/main/UPGRADE-GUIDE.md
[^3]: The Rack Specification (SPEC) — the canonical `call(env) → [status, headers, body]` contract. https://rack.github.io/rack/main/SPEC_rdoc.html
[^4]: Rack README, "Version support" and "Change log" sections. https://github.com/rack/rack#version-support

## Tags

ruby, web-server-interface, middleware, rack, wsgi, http, web-framework, gateway-interface
