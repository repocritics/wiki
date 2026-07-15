# sparklemotion/nokogiri

> The default way to parse, search, and build XML and HTML from Ruby — a thin, secure-by-default wrapper over native C and Java parsers.

[GitHub repo](https://github.com/sparklemotion/nokogiri) ·
[Official website](https://nokogiri.org/) ·
[License: MIT](https://github.com/sparklemotion/nokogiri/blob/main/LICENSE.md)

## Overview

Nokogiri (鋸, Japanese for "saw") is the de facto XML and HTML library for Ruby. It was created around 2008 by Aaron Patterson and Mike Dalessio and quickly displaced Hpricot as the community default[^1]. For most of a Ruby developer's career, "parse this HTML" means `require "nokogiri"`, and a large fraction of the scraping, feed-processing, SOAP, and templating code in the ecosystem depends on it transitively.

The project's own stated principles frame its two defining tradeoffs[^2]. First, it is **secure-by-default**: every document is treated as untrusted, so external entity expansion and network access are disabled unless you explicitly opt in. Second, it aims to be a **thin-as-reasonable layer** over the underlying native parsers rather than a portability shim. On CRuby that parser is libxml2 (plus libxslt for transforms and libgumbo for HTML5); on JRuby it is Xerces and NekoHTML. Nokogiri deliberately does *not* paper over behavioral differences between these backends, which means a document can parse slightly differently on CRuby versus JRuby, and the maintainers consider that acceptable rather than a bug.

The practical consequence is that Nokogiri is really a Ruby API bolted onto decades-old, security-sensitive C libraries. Most of its historical pain — slow installs, compiler toolchain failures, and a steady stream of libxml2 CVEs — flows directly from that architecture, and most of the last several years of maintenance work has gone into insulating users from it.

## Getting Started

```bash
gem install nokogiri
# or in a Gemfile:
#   gem "nokogiri"
```

On common platforms this pulls a precompiled "native gem" and installs in seconds with no compiler required (see Architecture)[^3].

```ruby
require "nokogiri"
require "open-uri"

doc = Nokogiri::HTML5(URI.open("https://example.com/"))

# CSS3 selectors
doc.css("article h2").each { |h| puts h.text }

# XPath 1.0
doc.xpath("//a[@href]").each { |a| puts a["href"] }

# Building a document with the Builder DSL
builder = Nokogiri::XML::Builder.new do |xml|
  xml.root { xml.item(id: 1) { xml.text "hello" } }
end
puts builder.to_xml
```

## Architecture / How It Works

Nokogiri is not one parser but a stable Ruby API layered over different native engines per runtime:

- **CRuby (MRI/YARV):** a C extension linked against libxml2 (parsing, XPath, DOM), libxslt (XSLT 1.0), and libgumbo (the WHATWG-compliant HTML5 parser, merged in from the `nokogumbo` gem). By default Nokogiri **vendors and compiles its own copies** of these libraries rather than using the ones on your system[^2].
- **JRuby:** a Java extension over Xerces and NekoHTML. The public API matches, but the underlying engine and its quirks differ.

Parsing surfaces come in several shapes: a DOM parser for XML, HTML4, and HTML5; a SAX parser for streaming; a push parser for incremental input; and a Builder DSL for construction. Searching is available through XPath 1.0 and through CSS3 selectors (with a few jQuery-like extensions) that are compiled down to XPath internally.

The most consequential architectural decision is the **native/precompiled gem** distribution introduced in the 1.11 era[^3]. Historically, installing Nokogiri meant compiling libxml2 and libxslt from source on your machine — the single most common source of "why won't Nokogiri install" support tickets in Ruby's history. Native gems ship prebuilt binaries for a fixed matrix of platforms (`x86_64`/`aarch64`/`arm` Linux with both glibc and musl, `x86_64`/`arm64` Darwin, `x64-mingw-ucrt` Windows, and a `java` gem for JRuby). On a supported platform, install is a few-second download; off it, you fall back to source compilation and need a C toolchain, Ruby headers, and system libraries.

Because Nokogiri vendors its own libxml2, the version of libxml2 you get is the version Nokogiri bundled — not whatever your OS ships. This is a feature (fast security patching, reproducible behavior) and a footgun (you can be running a different libxml2 than the rest of your system, and audits that scan system packages miss it).

## Production Notes

- **The system-vs-vendored libxml2 split is the recurring operational hazard.** By default you run Nokogiri's bundled libxml2. Passing `--use-system-libraries` (or setting `NOKOGIRI_USE_SYSTEM_LIBRARIES`) links against the OS copy instead, which some distros and security teams mandate. When you do this you inherit the system libxml2's version, bugs, and behavioral differences — the maintainers recommend system libxml2 `>= 2.9.2`, and `>= 2.12.0` in more recent releases. Mixed setups produce subtle parsing differences that are painful to debug.
- **libxml2 CVEs land on you.** Nokogiri's security posture is largely libxml2's security posture. When a libxml2 vulnerability is disclosed, Nokogiri ships a patch release bumping the vendored library; keeping Nokogiri current is a security requirement, not just a maintenance nicety. Teams pinning old versions for stability accumulate real CVE exposure.
- **Secure-by-default means XXE is off — until you turn it on.** External entity expansion, DTD loading, and network access are disabled by default. Code that enables `Nokogiri::XML::ParseOptions::NOENT` (often copied from Stack Overflow to "fix" entity handling) re-opens the classic XXE attack surface. Treat any `NOENT`/`DTDLOAD`/`NONET`-off change as a security review item.
- **Native gems assume a supported platform.** On an unlisted architecture, an Alpine/musl image without the right variant, or an air-gapped build, you drop back to source compilation and its full toolchain requirements. CI on exotic platforms should budget for compile time and possible failures.
- **Cross-runtime behavior differs by design.** If you deploy on both CRuby and JRuby, do not assume byte-identical parse output; whitespace handling, error recovery on malformed HTML, and namespace edge cases can diverge. Write assertions against semantic content, not exact serialization.
- **Memory and large documents.** DOM parsing holds the whole tree in native memory; very large XML is better handled with the SAX or push (`Reader`) APIs. Freed Nokogiri nodes are backed by C structures, so holding Ruby references to detached subtrees can keep native memory alive longer than the GC pressure suggests.

## When to Use / When Not

**Use when:**
- You need robust, standards-aware HTML5 or XML parsing in Ruby with mature XPath and CSS querying.
- You're scraping, transforming, or generating markup and want the ecosystem-default library with the most documentation and Stack Overflow coverage.
- You need XSLT 1.0, XSD schema validation, or a SAX/streaming path for large inputs.

**Avoid when:**
- You cannot ship or compile native code and are on an unsupported platform — a pure-Ruby parser avoids the toolchain entirely.
- You only need to touch tiny, trusted, well-formed XML and want zero native dependencies (stdlib REXML is enough).
- Your workload is HTML5-parsing-heavy and latency-critical; a Lexbor-based parser can be significantly faster for that specific case.

## Alternatives

- ruby/rexml — pure-Ruby XML parser in the standard library; use when you want zero native dependencies and can accept slower parsing on small, trusted documents.
- ohler55/ox — fast C-based XML parser and (de)serializer; use when you need raw XML throughput and don't need HTML or CSS/XPath querying.
- YorickPeterse/oga — pure-Ruby XML/HTML parser with XPath/CSS and no libxml2; use when you want thread-safety and to avoid the C dependency entirely.
- serpapi/nokolexbor — Nokogiri-compatible API backed by the Lexbor HTML5 engine; use when HTML5 parsing speed is the bottleneck and you want a near drop-in swap.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2008 | Initial release; libxml2-backed, positioned as the successor to Hpricot[^1]. |
| 1.11.0 | 2021-01 | Precompiled "native gems" for Linux/Darwin/Windows — largely ended the install-compilation era[^3]. |
| 1.12.0 | 2021-05 | HTML5 parsing merged in from `nokogumbo` (`Nokogiri::HTML5`, libgumbo). |
| 1.13.0 | 2022 | Broader native-gem platform matrix (musl, ARM) and libxml2 updates. |
| 1.16.0 | 2023-11 | Continued libxml2/libxslt bumps; raised minimum Ruby support. |
| 1.18.0 | 2024-12 | Ongoing dependency and platform maintenance. |

Nokogiri has never shipped a 2.0 — per its versioning policy it has "never done" a major bump, because the public API has stayed backward-compatible; dropping EOL Ruby versions and updating vendored libraries are treated as minor bumps[^2].

## References

[^1]: Nokogiri project background and history — https://nokogiri.org/ and repository README, https://github.com/sparklemotion/nokogiri
[^2]: Nokogiri README — Guiding Principles, Technical Overview (CRuby/JRuby backends), and Semantic Versioning Policy. https://github.com/sparklemotion/nokogiri/blob/main/README.md
[^3]: Nokogiri installation and native gems documentation. https://nokogiri.org/tutorials/installing_nokogiri.html

## Tags

ruby, xml, html, html5, parser, xpath, css-selectors, libxml2, xslt, sax, web-scraping, native-extension
