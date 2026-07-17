# zeux/pugixml

> A light-weight C++ XML library: a mutable DOM tree, a very fast in-place parser, and XPath 1.0 — in three source files.

[GitHub repo](https://github.com/zeux/pugixml) ·
[Official website](https://pugixml.org/) ·
[License: MIT](https://github.com/zeux/pugixml/blob/master/LICENSE.md)

## Overview

pugixml is a C++ XML processing library built around a DOM-like, fully mutable tree, a parser that constructs that tree from a file or buffer, and an XPath 1.0 engine for data-driven queries[^1]. It is maintained by Arseny Kapoulkine (zeux), who also authored meshoptimizer and volk. The code descends from Kristen Wegner's public-domain "pugxml" parser (2003); pugixml as a rewritten, XPath-capable library dates to 2010[^2].

The defining tradeoff is **DOM, not streaming**. pugixml loads the entire document into an in-memory tree, so memory use scales with document size (typically a few times the source bytes) and there is no SAX-style incremental event API. In exchange you get a small, dependency-free codebase, a genuinely fast parser, and an API that most C++ developers can use correctly within an hour. It occupies the niche of "I need to read/edit config- to medium-sized XML in C++ without pulling in libxml2 or Xerces."

pugixml deliberately does **not** implement DTD validation, XML Schema, or namespace-aware resolution beyond exposing raw prefixed names. It parses (and can skip) DOCTYPE declarations but performs no validation against them[^3]. If your problem requires conformance-grade XML processing, this is the wrong library, and the maintainer says so plainly in the manual.

## Getting Started

There is no mandatory build system: the library is three files — `pugixml.hpp`, `pugixml.cpp`, `pugiconfig.hpp` — that you drop into your project and compile alongside your own sources. It is also packaged for CMake, vcpkg, Conan, and most Linux distributions.

```bash
# vcpkg
vcpkg install pugixml
# or add the three source files directly to your build
```

```cpp
#include "pugixml.hpp"
#include <iostream>

int main() {
    pugi::xml_document doc;
    pugi::xml_parse_result result = doc.load_file("config.xml");
    if (!result) {
        std::cerr << "parse error: " << result.description()
                  << " at offset " << result.offset << "\n";
        return 1;
    }

    for (pugi::xml_node tool : doc.child("Profile").child("Tools").children("Tool")) {
        int timeout = tool.attribute("Timeout").as_int();
        if (timeout > 0)
            std::cout << tool.attribute("Filename").value()
                      << " timeout=" << timeout << "\n";
    }
}
```

The same selection with XPath:

```cpp
pugi::xpath_node_set tools =
    doc.select_nodes("/Profile/Tools/Tool[@Timeout > 0]");
for (pugi::xpath_node n : tools)
    std::cout << n.node().attribute("Filename").value() << "\n";
```

## Architecture / How It Works

The parser reads the whole document into a single contiguous buffer and does **in-place parsing**: rather than allocating a string per node, it mutates that buffer (inserting terminators, decoding entities and encodings in place) and stores pointers into it[^1]. Tree nodes are small fixed-size structures allocated from internal memory pools, not individually `new`-ed. This is the core reason it is fast and low-overhead — allocation is amortized and string data is not copied twice.

`xml_document` owns everything; `xml_node` and `xml_attribute` are lightweight handles (essentially tagged pointers) that are cheap to copy and become dangling once the owning document is destroyed. The tree is fully mutable: you can append, remove, and reorder nodes and attributes, then serialize back out with configurable indentation and encoding.

**Unicode.** pugixml supports UTF-8, UTF-16, UTF-32, and Latin-1 input, auto-detecting encoding and converting during parse. The character type is selectable at compile time via `PUGIXML_WCHAR_MODE` (char vs. wchar_t builds), which is a global ABI decision, not a per-call one.

**XPath.** A standalone XPath 1.0 implementation compiles query strings into an expression tree (`xpath_query`) that can be reused across evaluations. It is string-typed like XPath 1.0 itself — no XPath 2.0/3.1 sequences, no schema-aware typing. XPath support can be compiled out entirely (`PUGIXML_NO_XPATH`) to shrink the binary.

**Configuration** happens through `pugiconfig.hpp` macros: disabling exceptions (`PUGIXML_NO_EXCEPTIONS`), STL (`PUGIXML_NO_STL`), or XPath; forcing an API version namespace; injecting custom allocators. These are compile-time switches, so mixing translation units built with different `pugiconfig.hpp` settings is an ODR hazard.

## Production Notes

- **Memory, not streaming.** The entire document lives in RAM as a tree. For multi-hundred-MB or GB documents this is disqualifying — reach for a pull/SAX parser (libxml2's reader, or expat) instead. There is no incremental parse API; `load_buffer_inplace` lets you hand pugixml your own buffer to avoid one copy, but the tree still materializes fully.
- **Handle lifetime.** `xml_node`/`xml_attribute` are non-owning. Returning them past the lifetime of their `xml_document`, or after `reset()`/reload, gives dangling access with no diagnostic. Store the document, pass the handles.
- **Encoding surprises.** Auto-detection keys off the BOM and XML declaration; a mislabeled file (declared UTF-8, actually Latin-1) parses without error and yields mojibake rather than a failure. Validate encoding upstream if inputs are untrusted.
- **`wchar_t` mode is contagious.** Building with `PUGIXML_WCHAR_MODE` changes the string type across the whole API and doubles/quadruples string storage on some platforms. Decide once, per project.
- **No validation is a security posture, not a gap.** Because pugixml does not process external DTD entities, it is not vulnerable to classic XXE / billion-laughs entity-expansion attacks the way validating parsers are — a real advantage for untrusted input. It still reads the whole document into memory, so guard against oversized inputs yourself.
- **Thread safety.** Distinct documents are independent and can be parsed on separate threads. A single document is not internally synchronized; concurrent mutation needs external locking. XPath `xpath_query` objects are immutable after compilation and safe to share for read.
- **Stability.** The API has been stable for years; upgrades across minor versions rarely break source. The maintainer keeps an explicit deprecation policy and the master branch is the released code — there is no long-lived unstable branch to track.

## When to Use / When Not

**Use when:**
- You need to read, edit, and write config- to medium-sized XML in C++ with minimal fuss.
- You want zero third-party dependencies and a drop-in three-file build.
- You need XPath 1.0 queries over an in-memory tree.
- You're parsing untrusted XML and want no DTD/entity attack surface.

**Avoid when:**
- Documents are too large to hold in memory, or you need streaming/incremental parsing.
- You require DTD/Schema validation, XML namespaces resolution, or XPath 2.0+/XSLT.
- You need canonicalization, XML signatures, or other W3C conformance features.
- You're in a language other than C++ (there are idiomatic parsers everywhere else).

## Alternatives

- GNOME/libxml2 — the reference C library; validation, namespaces, XPath, streaming reader. Use it when you need conformance or huge-document streaming and can accept a heavier dependency and C API.
- libexpat/libexpat — pure streaming (SAX/push) parser. Use it when memory is the constraint and you don't need a tree.
- leethomason/tinyxml2 — even smaller two-file DOM parser, no XPath, no Unicode conversions. Use it when you want the absolute minimum and can drop XPath.
- Tencent/rapidxml (RapidXML lineage) — the in-place parser pugixml is often benchmarked against; faster in narrow cases but a thinner, less maintained API. Use it only for raw parse speed with no editing/XPath needs.
- USCiLab/cereal or nlohmann/json — if XML is incidental and you control the format, a different serialization format is often the better answer.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010-11 | First pugixml release: rewritten parser + XPath 1.0[^2]. |
| 1.7 | 2015-10 | Compact memory mode, custom allocators maturing. |
| 1.8 | 2016-11 | Long-term stable API baseline. |
| 1.9 | 2018-04 | Parser/serializer refinements. |
| 1.10 | 2019-09 | Build and packaging improvements. |
| 1.11 | 2020-11 | CMake install/config modernization. |
| 1.12 | 2022-02 | Bug fixes; 1.12.1 follow-up. |
| 1.13 | 2022-11 | Maintenance release. |
| 1.14 | 2023-10 | Maintenance release. |
| 1.15 | 2025-01 | Maintenance release. |
| 1.16 | 2026-06 | Latest release[^4]. |

## References

[^1]: pugixml manual — parsing, DOM, and in-place model. https://pugixml.org/docs/manual.html
[^2]: pugixml quick-start guide and project history (derived from Kristen Wegner's pugxml, 2003). https://pugixml.org/docs/quickstart.html
[^3]: pugixml manual — DOCTYPE handling and the absence of DTD/Schema validation. https://pugixml.org/docs/manual.html#loading.errors
[^4]: pugixml releases. https://github.com/zeux/pugixml/releases

## Tags

cpp, xml, xml-parser, dom, xpath, parser, serialization, header-library, unicode, cross-platform
