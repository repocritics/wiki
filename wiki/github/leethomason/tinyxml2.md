# leethomason/tinyxml2

> A two-file, dependency-free C++ DOM parser for XML — small, fast, and deliberately incomplete.

[GitHub repo](https://github.com/leethomason/tinyxml2) ·
[Documentation](http://leethomason.github.io/tinyxml2/) ·
[License: Zlib](https://github.com/leethomason/tinyxml2/blob/master/LICENSE.txt)

## Overview

TinyXML-2 is a C++ XML parser that ships as exactly one header (`tinyxml2.h`) and one source file (`tinyxml2.cpp`). It parses a document into a Document Object Model — a tree of C++ objects you can read, mutate, and serialize back out. It is a ground-up rewrite of the older TinyXML-1, retuned "for use in a game": fewer allocations, less memory, and no dependency on the C++ Standard Library, exceptions, or RTTI[^1]. First committed in 2012 by Lee Thomason, it has become one of the default embedded XML parsers in the C++ world, vendored directly into countless game engines, tools, and firmware projects.

The defining decision is scope. TinyXML-2 does **not** implement DTDs, XML Schema, XSLT, XPath, or namespaces, and it will not validate your document against anything[^1]. It reads UTF-8 only. This is not a limitation the project plans to fix — it is the product. The bet is that most programs storing application data in XML need a small correct DOM and nothing more, and that the cost of a full-featured parser (libxml2, Xerces) is rarely justified. If you need standards compliance or query languages, the README itself tells you to use a different parser.

The second tension is API stability. The project uses semantic versioning but warns that the major version "will (probably) change fairly rapidly" because "API changes are fairly common"[^1]. Vendoring the two files pins you to a known version, which is how most people avoid the churn.

## Getting Started

The intended install is copy-in: drop `tinyxml2.cpp` and `tinyxml2.h` into your project and compile them with your own sources. Package managers also carry it:

```sh
# vcpkg
vcpkg install tinyxml2
# or apt / brew
apt-get install libtinyxml2-dev
brew install tinyxml2
```

A minimal load-and-read, plus building a document from scratch:

```cpp
#include "tinyxml2.h"
using namespace tinyxml2;

int main() {
    XMLDocument doc;
    if (doc.LoadFile("config.xml") != XML_SUCCESS)
        return doc.ErrorID();

    // Navigate. NOTE: no null-checking here — a missing node segfaults.
    const char* title =
        doc.FirstChildElement("PLAY")->FirstChildElement("TITLE")->GetText();

    int volume = 0;
    doc.FirstChildElement("PLAY")
       ->QueryIntAttribute("volume", &volume);   // typed, error-returning

    // Build and save a new document.
    XMLDocument out;
    XMLElement* root = out.NewElement("root");    // owned by `out`, do not delete
    root->SetAttribute("count", 3);
    out.InsertFirstChild(root);
    out.SaveFile("out.xml");
    return 0;
}
```

Use `XMLHandle` when you want null-safe traversal instead of hand-checking every pointer.

## Architecture / How It Works

Everything hangs off `XMLNode`. `XMLDocument`, `XMLElement`, `XMLText`, `XMLComment`, `XMLDeclaration`, and `XMLUnknown` all derive from it; attributes (`XMLAttribute`) are a separate linked list hung off each element. There is no `dynamic_cast` — because RTTI is off, downcasting goes through explicit `ToElement()` / `ToText()` methods that return `nullptr` on mismatch.

Two internal mechanisms explain the speed and the low allocation count:

- **In-place parsing via `StrPair`.** The parser does not copy strings out of the input as it goes. `XMLDocument` owns one heap buffer holding the whole document text, and nodes hold `StrPair` references *into* that buffer. Entity decoding and null-termination are done by mutating the buffer in place. This is why the document owns the character data and why nodes are only valid for the lifetime of their `XMLDocument`.
- **Fixed-block memory pools (`MemPoolT`).** Nodes and attributes are not `new`'d individually; they come from per-type pools that allocate in large blocks. This collapses thousands of small allocations into a handful, which matters most on allocator-constrained platforms (consoles, embedded).

Ownership is centralized: any node must be created through `XMLDocument::NewElement` / `NewText` / etc., is owned by the document, and is destroyed when the document is destroyed[^1]. You hold raw pointers but you do not `delete` them. Traversal can be done by walking `FirstChildElement` / `NextSiblingElement`, or via the `XMLVisitor` pattern (used internally by the printer). Output goes through `XMLPrinter`, which can serialize an `XMLDocument`, print to a `FILE*` or memory buffer, or stream elements directly without ever building a DOM.

## Production Notes

**In-memory DOM only.** The entire document and its node tree live in RAM. There is no streaming/SAX input mode — `LoadFile` reads the whole file first. For multi-hundred-MB or GB XML you want an event parser (Expat, libxml2 SAX) instead; TinyXML-2's memory footprint scales with the whole document plus node overhead.

**Convenience getters are null-unsafe.** A `->FirstChildElement("X")->GetText()` chain returns `nullptr` on any missing link and will crash on the next dereference. The README's own example flags this "dangerous lack of error checking"[^1]. In real code, prefer `QueryIntAttribute` / `QueryDoubleAttribute` (which return an `XMLError` and leave your variable untouched on failure) and `XMLHandle` for navigation.

**Whitespace is lossy by design.** With the default `PRESERVE_WHITESPACE`, text *inside* an element is kept but whitespace *between* elements is discarded — a pretty-printed and a minified document parse identically[^1]. `COLLAPSE_WHITESPACE` gives HTML-like behavior but the README notes it currently parses the document twice, so there is a real performance cost. `PEDANTIC_WHITESPACE` keeps inter-element text but is newer and less battle-tested.

**Not thread-safe per document.** Concurrent mutation of one `XMLDocument` is unsafe; the usual pattern is one document per thread. There is no shared-tree locking.

**Entity round-tripping is imperfect.** The five predefined entities and numeric character references are decoded on read, but a numeric reference like `&#160;` is written back as the raw code point — correct output, but the original entity syntax is not preserved[^1].

**API churn across majors.** Because "API changes are fairly common," upgrading across major versions can break call sites. The dominant mitigation is to vendor the two files at a fixed version rather than track a system package. Build systems other than the bundled CMake have been removed over time to reduce maintenance surface[^1].

## When to Use / When Not

**Use when:**
- You want a small, embeddable DOM parser with zero external dependencies.
- You control the XML (config files, save games, tool interchange) and don't need validation.
- You're on a platform where allocation count and binary size matter (games, embedded, consoles) and STL/exceptions/RTTI are unwelcome.
- You want trivial vendoring — two files, drop them in, done.

**Avoid when:**
- You need DTD/XSD validation, XPath queries, XSLT, or namespaces — none exist.
- You must parse very large documents with a bounded memory budget (use a SAX parser).
- You need UTF-16/wide-char input, or lenient HTML-style parsing.
- You need a frozen ABI — the project reserves the right to change the API on major bumps.

## Alternatives

- zeux/pugixml — light DOM parser with XPath 1.0 support and generally faster parsing; use when you need queries or peak speed and can accept STL usage.
- GNOME/libxml2 — the reference C implementation with DTD/XSD/XPath/SAX; use when you need standards compliance or streaming over huge files.
- libexpat/libexpat — stream-oriented (SAX) C parser; use when documents are too large for a DOM or you want event-driven parsing.
- RapidXML — extreme-speed header-only in-situ parser; use when read throughput dominates and you don't need mutation/save ergonomics.
- boostorg/property_tree — use when you're already in Boost and want a simple config tree rather than a real XML DOM.

## History

| Version | Date | Notes |
|---------|------|-------|
| TinyXML-1 | pre-2012 | Predecessor by Lee Thomason; key contributors Yves Berquin, Andrew Ellerton[^1]. |
| initial | 2012-02 | TinyXML-2 rewrite: game-oriented, no STL/RTTI/exceptions, memory pools[^2]. |
| 3.0.0 | 2015-03 | Semantic-versioning era; GitHub-tagged releases begin. |
| 5.0.0 | 2017-06 | — |
| 7.0.0 | 2018-11 | — |
| 8.0.0 | 2020-03 | — |
| 9.0.0 | 2021-06 | — |
| 10.0.0 | 2023-12 | — |
| 11.0.0 | 2025-03 | Latest major[^3]. |

## References

[^1]: TinyXML-2 README — features, scope, memory model, whitespace, license. https://github.com/leethomason/tinyxml2/blob/master/readme.md
[^2]: Repository creation and initial TinyXML-2 commit (2012-02-25). https://github.com/leethomason/tinyxml2
[^3]: TinyXML-2 GitHub Releases (10.0.0 2023-12-31, 10.1.0 2025-03-08, 11.0.0 2025-03-15). https://github.com/leethomason/tinyxml2/releases

## Tags

cpp, xml, xml-parser, dom, serialization, parser, embedded, header-library, zlib-license, no-dependencies, game-development
