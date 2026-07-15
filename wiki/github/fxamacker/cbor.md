# fxamacker/cbor

> A CBOR (RFC 8949) codec for Go with an `encoding/json`-shaped API and security-first defaults.

[GitHub repo](https://github.com/fxamacker/cbor) ·
[License: MIT](https://github.com/fxamacker/cbor/blob/master/LICENSE)

## Overview

`fxamacker/cbor` encodes and decodes CBOR — Concise Binary Object Representation, an IETF Internet Standard (STD 94 / RFC 8949, formerly RFC 7049)[^1]. CBOR is a self-describing binary format positioned as a compact alternative to JSON: it carries the same data model (maps, arrays, strings, numbers, booleans, null) but in fewer bytes and without a text-parsing step. The library also handles CBOR Sequences (RFC 8742) and Extended Diagnostic Notation (RFC 8610 Appendix G)[^2].

The project's defining choice is that it optimizes for **safe decoding of untrusted input** before speed. The decoder ships with configurable limits (max nested levels, max array/map elements, max byte-string length) and optional duplicate-map-key detection, so malformed or adversarial CBOR is rejected cheaply rather than triggering unbounded allocation. This is the practical differentiator versus Go's `encoding/gob`, which is explicitly not hardened against adversarial input, and versus some older third-party codecs that could exhaust memory on hostile data.

It is a widely embedded low-level dependency rather than an application framework. The README lists production use inside Kubernetes, Microsoft, Let's Encrypt, Tailscale, Red Hat/OpenShift, IBM, Arm, and the Veraison/COSE attestation stack[^3] — CBOR is the wire format underneath WebAuthn/FIDO2, COSE, CWT, and various IoT and attestation protocols, and this is the dominant pure-Go implementation for them.

## Getting Started

```bash
go get github.com/fxamacker/cbor/v2
```

```go
package main

import (
	"fmt"

	"github.com/fxamacker/cbor/v2"
)

type Coord struct {
	X int `cbor:"x"`
	Y int `cbor:"y"`
}

func main() {
	// Default mode: package-level funcs mirror encoding/json.
	b, _ := cbor.Marshal(Coord{X: 1, Y: 2}) // []byte, 9 bytes
	fmt.Printf("%x\n", b)                    // a2617801617902

	var c Coord
	_ = cbor.Unmarshal(b, &c)
	fmt.Printf("%+v\n", c) // {X:1 Y:2}

	text, _ := cbor.Diagnose(b) // human-readable diagnostic notation
	fmt.Println(text)           // {"x": 1, "y": 2}
}
```

The import path is versioned (`/v2`); the `v1` line exists but `v2` is the maintained module.

## Architecture / How It Works

The API deliberately mirrors `encoding/json`: `Marshal`/`Unmarshal`, `NewEncoder`/`NewDecoder`, and struct tags. Migrating serialization from JSON is largely a matter of swapping the package and the `json:` tags for `cbor:`. Reflection drives struct encoding the same way.

The central abstraction is the **mode**. Rather than mutating global state, you build an immutable `EncMode` or `DecMode` from an options struct at startup and reuse it across goroutines:

```go
opts := cbor.CoreDetEncOptions() // preset: RFC 8949 Core Deterministic Encoding
opts.Time = cbor.TimeUnix        // adjust individual settings
em, _ := opts.EncMode()          // immutable, concurrency-safe
b, _ := em.Marshal(v)
```

Modes are immutable once created, which is what makes them safe for concurrent use without locking. Presets cover the canonical serialization profiles that protocols require: `CoreDetEncOptions` (Core Deterministic Encoding), `PreferredUnsortedEncOptions` (Preferred Serialization), `CanonicalEncOptions` (RFC 7049 Canonical), and `CTAP2EncOptions` (FIDO2 CTAP2 Canonical CBOR used by WebAuthn). Getting these byte-exact matters because deterministic encoding is a correctness requirement for signatures and content addressing, not a nicety.

**Struct tag options** trade schema-coupling for size. `toarray` drops field names entirely and encodes a struct as a positional CBOR array; `keyasint` encodes field names as small integers; `omitempty` and `omitzero` drop absent fields; `-` omits a field. These are how CBOR-based protocols (COSE, CWT) get their compact integer-keyed maps and arrays without hand-rolling encoders.

**CBOR tags** (the format's extension mechanism) are handled via a `TagSet` that maps tag numbers to Go types, wired into a mode with `EncModeWithTags`/`DecModeWithTags`. For tag numbers the library doesn't model directly, user types can implement `cbor.Marshaler`/`cbor.Unmarshaler` (`MarshalCBOR`/`UnmarshalCBOR`), which the codec calls automatically — the same escape hatch pattern as `json.Marshaler`.

The implementation avoids Go's `unsafe` package; faster-but-riskier behaviors are opt-in rather than default. Encoding optionally shrinks `float64`→`float32`→`float16` when a value round-trips losslessly.

## Production Notes

**Decoder limits are the whole point — tune them deliberately.** Defaults are conservative (e.g. `MaxNestedLevels` of 32). If you decode legitimately deep or large structures you must raise the relevant limit on a `DecMode`; if you accept untrusted input, keep limits tight and enable `DupMapKeyEnforcedAPF` for duplicate-key rejection. Leaving these at defaults is the safe failure direction, but silent truncation-by-error surprises teams migrating from permissive codecs.

**Build modes once.** Creating an `EncMode`/`DecMode` per call defeats the design and adds reflection setup cost each time. The intended pattern is package-level modes constructed at startup. Because they're immutable there is no reason not to share them.

**CBOR is not self-versioning.** Choosing `toarray` or `keyasint` bakes field order / integer keys into the wire format, so reordering or renumbering struct fields is a breaking change to the encoding — plan field layout the way you would a protobuf schema, not a JSON object.

**Deterministic encoding is preset-specific.** Different CBOR libraries and different protocols use different canonicalization rules (RFC 7049 Canonical vs. RFC 8949 Core Deterministic vs. CTAP2). Interop bugs almost always trace back to two sides using different presets. Pick the profile the protocol mandates rather than the default.

**`float16` shrinking can surprise consumers.** Optional half-float encoding minimizes size but assumes the decoder handles all three float widths (this library does). A non-conformant peer decoder may not.

**TinyGo support is a separate, experimental branch** (`feature/cbor-tinygo-beta`), not the main module, with a reduced default nesting limit and no fuzz coverage — treat embedded/TinyGo use as beta.

Maintenance is active: the project reports >97% test coverage, continuous fuzzing, CodeQL scanning, and passed confidential third-party security assessments in 2022 (one non-confidential NCC Group report for Microsoft's go-cose is public)[^4].

## When to Use / When Not

**Use when:**
- You need a spec-conformant CBOR codec in pure Go (COSE, CWT, WebAuthn/FIDO2, EdgeX, IoT, attestation).
- You decode untrusted input and want configurable resource limits and duplicate-key detection.
- You want deterministic/canonical encoding for signing or content addressing.
- You're migrating from `encoding/json` and want a near-identical API with smaller output.

**Avoid when:**
- Your peers speak a different binary format — pick that format's library instead (MessagePack, Protocol Buffers).
- You need human-readable, log-greppable, or browser-native payloads — plain JSON stays easier to debug.
- You're Go-to-Go only and never cross a trust or language boundary — `encoding/gob` is simpler (but do not expose it to untrusted input).

## Alternatives

- ugorji/go — multi-format codec (CBOR, MessagePack, JSON, Binc) with code-gen; use when you need several wire formats behind one API, but review decoder-hardening behavior on untrusted input.
- vmihailenco/msgpack — mature Go MessagePack codec; use when your protocol is MessagePack rather than CBOR.
- tinylib/msgp — code-generated MessagePack for Go; use when you want generated (non-reflection) encoders for maximum throughput and are on MessagePack.
- protocolbuffers/protobuf (or google/flatbuffers) — schema-first IDL formats; use when you want enforced schemas and cross-language codegen rather than a self-describing schemaless format.
- CBOR-specific: the older brianolson/cbor_go predates this library and is effectively unmaintained; new Go projects should default here.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2019-05 | Initial release, ~2 months after repo creation. |
| v2.0.0 | 2020-02 | v2 module: immutable modes, options structs. |
| v2.3.0 | 2021-05 | Struct tag options and TagSet API maturation. |
| v2.4.0 | 2022-01 | Stabilized v2 feature set. |
| v2.5.0 | 2023-08 | `UnmarshalFirst` / `DiagnoseFirst` for CBOR Sequences. |
| v2.6.0 | 2024-02 | Decoder/option additions. |
| v2.7.0 | 2024-06 | `MarshalToBuffer` + `UserBufferEncMode` (user-supplied buffers). |
| v2.8.0 | 2025-03 | Maintenance and option refinements. |
| v2.9.0 | 2025-07 | Continued feature/hardening line. |
| v2.9.2 | 2026-05 | Latest tagged release as of writing. |

## References

[^1]: IETF STD 94 / RFC 8949, "Concise Binary Object Representation (CBOR)." https://www.rfc-editor.org/info/std94
[^2]: RFC 8742, "CBOR Sequences." https://www.rfc-editor.org/rfc/rfc8742.html
[^3]: fxamacker/cbor README, "Who uses fxamacker/cbor." https://github.com/fxamacker/cbor#who-uses-fxamackercbor
[^4]: NCC Group security assessment of go-cose (subset) for Microsoft, 2022-05-26. https://github.com/veraison/go-cose/blob/v1.0.0-rc.1/reports/NCC_Microsoft-go-cose-Report_2022-05-26_v1.0.pdf

## Tags

go, golang, cbor, rfc-8949, serialization, codec, binary-format, cose, webauthn, security, json-alternative
