# jinzhu/copier

> Reflection-based struct-to-struct field copier for Go, matching by field and method name.

[GitHub repo](https://github.com/jinzhu/copier) ·
[License: MIT](https://github.com/jinzhu/copier/blob/master/License)

## Overview

Copier is a small Go utility that copies values between arbitrary types at runtime using reflection. Its core promise is name-based mapping: a field on the destination is populated from a source field (or a source method) with the same name, without you writing the assignment by hand. It also handles slice-to-slice, struct-to-slice, and map-to-map copying, plus a handful of tag directives for must-copy, ignore, and override semantics.

The library comes from Jinzhu Zhang, the author of GORM, and shares that project's design instinct: heavy use of reflection to trade raw speed for ergonomic, convention-driven code. The repository predates Go generics — it was created in 2013[^1] and only received its first semver tag (v0.1.0) in late 2020[^2] — so its API is built entirely around `interface{}`/`any` and runtime type inspection rather than compile-time type parameters.

The defining tension is exactly that reflection dependency. Copier removes boilerplate DTO-mapping code, which is genuinely useful at the boundary between database models, API request/response structs, and gRPC messages. But it does so by moving field mismatches from compile time to runtime, and by paying a reflection cost on every call. Whether that trade is worth it depends heavily on how hot the copy path is and how much you value a compiler-checked mapping.

## Getting Started

```bash
go get -u github.com/jinzhu/copier
```

```go
package main

import (
	"fmt"

	"github.com/jinzhu/copier"
)

type User struct {
	Name string
	Age  int32
	Role string
}

// A method whose name matches a destination field becomes a copy source.
func (u User) DoubleAge() int32 { return 2 * u.Age }

type Employee struct {
	Name      string
	Age       int32
	DoubleAge int32 // filled from User.DoubleAge()
	Role      string
}

func main() {
	user := User{Name: "Jinzhu", Age: 18, Role: "Admin"}
	var employee Employee

	if err := copier.Copy(&employee, &user); err != nil {
		panic(err)
	}
	fmt.Printf("%#v\n", employee)
	// Employee{Name:"Jinzhu", Age:18, DoubleAge:36, Role:"Admin"}
}
```

Destination must be a pointer. For finer control, `CopyWithOption` takes a `copier.Option{IgnoreEmpty, DeepCopy, CaseSensitive, Converters, ...}`.

## Architecture / How It Works

Copier is a single-package library with no external dependencies. The entire mechanism is `reflect`: given two values, it walks the destination type's fields, and for each one looks for a match on the source — first a field of the same name, then a nilary method of the same name whose single return value is assignable. Matches are resolved by name string comparison (case-insensitive by default; `CaseSensitive` opts into exact matching).

Copying rules are largely driven by assignability and kind:

- **Same-name fields** are set directly when the types are assignable, or recursively copied when both are structs.
- **Methods to fields** let computed values (`DoubleAge()`) flow into the destination, and — going the other direction — a source field can feed a destination setter method.
- **Slices and maps** are iterated element by element; each element goes through the same copy logic, which is how struct-to-slice and slice-to-slice work.
- **Tags** on destination fields adjust behavior: `copier:"-"` skips a field, `copier:"must"` forces a copy (panicking, or returning an error with `nopanic`), `copier:"override"` copies even when `IgnoreEmpty` would otherwise skip a zero/nil source, and a bare `copier:"OtherName"` remaps a differently named source field.

Two options change the depth of the operation. By default copies are shallow: pointers, slices, and maps are assigned by reference, so the destination shares backing storage with the source. `DeepCopy: true` instead allocates and recursively clones, at additional reflection and allocation cost. `Converters` let you register explicit `func(src) (dst, error)` transforms for type pairs the reflection walk cannot bridge (for example `time.Time` to `string`), which is the escape hatch for the many cases name-matching alone cannot express.

## Production Notes

**Silent no-ops are the primary footgun.** If a destination field has no matching source field or method, Copier simply leaves it at its zero value — no error, no warning. Rename a field on one side and the copy quietly stops happening. Because the mapping is by string name, there is nothing the compiler can catch. Teams that adopt Copier widely tend to backstop it with tests that assert specific fields are populated, since the library itself will not tell you a mapping broke.

**Reflection cost is real on hot paths.** Every call re-inspects types via `reflect`; there is no generated code. For occasional boundary mapping this is irrelevant, but in a tight loop or a high-QPS request handler, a hand-written assignment (or a code-generated mapper) can be an order of magnitude faster and allocate far less. Benchmark before using it inside anything latency-sensitive.

**Shallow-by-default aliasing bites.** Without `DeepCopy`, copied slices, maps, and pointers are shared with the source. Mutating the destination's slice can mutate the source's, which is a subtle source of data races and action-at-a-distance bugs. Reach for `DeepCopy: true` when the two values must be independent — and accept the extra allocation that implies.

**`IgnoreEmpty` interacts non-obviously with `override`.** `IgnoreEmpty: true` skips zero-valued source fields, which is convenient for partial updates, but it means a legitimately-zero value (empty string, `false`, `0`) will not overwrite the destination unless the field is tagged `copier:"override"`. Getting "update to empty" semantics right requires knowing this pairing.

**Maintenance is real but slow-cadence.** The repository is not archived and still receives commits (last push 2026-03-14), but tagged releases are infrequent — v0.4.0 (Aug 2023) is the most recent tag[^2], and the library sits below v1.0.0. Treat the master branch as the source of truth for recent fixes, and pin a commit or tag explicitly rather than tracking `latest`.

## When to Use / When Not

**Use when:**
- You map between structurally similar structs at system boundaries (model to DTO, request to entity) and the mapping is not on a hot path.
- Field names line up well and you want to delete repetitive assignment code.
- You need method-to-field or field-to-method plumbing that plain assignment cannot express concisely.

**Avoid when:**
- The copy runs in a tight loop or high-throughput handler where reflection cost and allocations matter — prefer hand-written code or a codegen mapper.
- You want compile-time guarantees that every field is mapped; Copier's name-matching fails silently.
- The transformation is complex enough that you would spend more time wiring `Converters` and tags than writing explicit code.

## Alternatives

- jmattheis/goverter — use when you want the same DTO-mapping ergonomics but with compile-time-checked, code-generated mappers instead of reflection.
- ulule/deepcopier — use when you want a similar tag-driven struct copier and are comparing ecosystem options with a comparable API surface.
- tiendc/go-deepcopy — use when your real need is deep cloning of a value graph rather than name-matched field mapping.
- google/go-cmp — not a copier, but use when the actual task is comparing two structs (typically in tests) rather than copying one into another.
- Hand-written assignment with Go generics — use when the copy is hot, security-sensitive, or simple enough that explicit code is clearer than any library.

## History

| Version | Date | Notes |
|---------|------|-------|
| (initial) | 2013-10-31 | Repository created; imported by commit, no semver tags[^1]. |
| v0.1.0 | 2020-11-27 | First tagged release[^2]. |
| v0.2.0 | 2021-01-10 | Options and tag-directive maturation. |
| v0.3.0 | 2021-04-22 | Continued option/tag work through the v0.3.x line. |
| v0.4.0 | 2023-08-09 | Most recent tag; library remains pre-1.0[^2]. |

## References

[^1]: GitHub API — `repos/jinzhu/copier`, `created_at` 2013-10-31, license MIT, ~6.2k stars / ~494 forks as of retrieval. https://github.com/jinzhu/copier
[^2]: GitHub API — `repos/jinzhu/copier/tags`; tag commit dates via the Git refs/commits API. First tag v0.1.0 (2020-11-27), latest tag v0.4.0 (2023-08-09). https://github.com/jinzhu/copier/tags

## Tags

go, golang, struct-mapping, reflection, dto, object-copy, deep-copy, boilerplate-reduction, utility-library, jinzhu
