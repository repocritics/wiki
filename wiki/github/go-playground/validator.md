# go-playground/validator

> Struct and field validation for Go driven by tags — cross-field, cross-struct, and deep dives into slices, maps, and arrays.

[GitHub repo](https://github.com/go-playground/validator) ·
[License: MIT](https://github.com/go-playground/validator/blob/master/LICENSE)

## Overview

`validator` is the de facto validation library for Go structs. You annotate struct fields with `validate:"..."` tags and call `validate.Struct(v)`; the library walks the value via reflection and returns a `ValidationErrors` slice describing every field that failed[^1]. It has been the default binding validator in the Gin web framework for years, which is why most Go request-handling code has a transitive dependency on it even when the author never imported it directly[^2].

The library's reach comes from breadth: it ships well over a hundred built-in tags covering comparisons (`gt`, `lte`, `oneof`), string shapes (`alphanum`, `email`, `uuid4`), network formats (`ip`, `cidr`, `fqdn`, `hostname_rfc1123`), and a long tail of format checks (`iso3166_1_alpha2`, `credit_card`, `bic`, `jwt`, `semver`). Cross-field tags (`eqfield`, `gtfield`) and cross-struct relative tags (`eqcsfield`) let one field's validity depend on another's, and the `dive` keyword recurses into collection elements and map keys.

The defining tension is that all of this rides on struct tags and runtime reflection. Rules are strings resolved at runtime, so a typo in a tag or an unregistered custom tag is not a compile error — it is a runtime panic or a silently skipped rule. You trade Go's compile-time guarantees for declarative density. Whether that trade is worth it is the recurring argument between this library and its programmatic-builder alternatives.

The project is mature but thinly staffed: the maintainer has posted an open call for co-maintainers, and issue throughput reflects that[^3]. It is not abandoned — commits and releases continue as of 2026 — but expect slow review on non-trivial PRs.

## Getting Started

```bash
go get github.com/go-playground/validator/v10
```

```go
package main

import (
	"errors"
	"fmt"

	"github.com/go-playground/validator/v10"
)

type SignupForm struct {
	Email    string `validate:"required,email"`
	Age      uint8  `validate:"gte=13,lte=130"`
	Password string `validate:"required,min=8"`
	Confirm  string `validate:"eqfield=Password"`
	Role     string `validate:"oneof=admin editor viewer"`
}

func main() {
	// Reuse a single instance — New() caches per-struct tag metadata.
	validate := validator.New(validator.WithRequiredStructEnabled())

	err := validate.Struct(SignupForm{Email: "not-an-email", Age: 9})
	var verrs validator.ValidationErrors
	if errors.As(err, &verrs) {
		for _, e := range verrs {
			fmt.Printf("%s failed on rule %q\n", e.Field(), e.Tag())
		}
	}
}
```

Validation functions return `error` to avoid an interface-typing pitfall where a typed nil is never equal to `nil`[^4]. In practice you check `err != nil` and, if you need per-field detail, type-assert (via `errors.As`) to `ValidationErrors`.

## Architecture / How It Works

`validator.New()` returns a `*Validate` that lazily builds and caches a `cStruct` (a compiled description of a type's fields and their parsed tags) the first time it sees each struct type. Subsequent validations of the same type reuse the cached plan, so steady-state cost is dominated by reflection on field values, not tag parsing. Simple field checks run in tens of nanoseconds with zero allocations on the success path; failures allocate to build the error objects[^5].

Tag strings are parsed into an ordered list of checks. Comma separates AND'd rules; `|` expresses OR within a rule; `dive` shifts subsequent tags down one level into slice/array/map elements, and `keys`/`endkeys` scope tags to map keys. This mini-grammar is powerful but positional — the meaning of a tag depends on how many `dive`s precede it — which is the source of most "why isn't my nested rule firing" confusion.

Extension points:

- **`RegisterValidation(tag, fn)`** — add a custom tag backed by a Go function. Must be registered before concurrent use; the instance is only safe for concurrent `Struct` calls once all registration is done.
- **`RegisterStructValidation`** — whole-struct rules that need multiple fields at once, kept out of tag strings.
- **`RegisterTagNameFunc`** — remap `Field()` to report, e.g., the `json` tag name instead of the Go field name, so API error messages match wire names.
- **Translations** — `FieldError` carries the tag and params but not a human sentence. The companion `universal-translator` and `locales` packages, plus per-locale registration, turn errors into i18n strings. This is deliberately not automatic and is a common setup surprise.

`*Validate` is safe for concurrent use after setup because the type cache uses a read-mostly lock. The intended pattern is one process-wide instance.

## Production Notes

- **Reflection cost is real at high fan-out.** Per-call overhead is small, but validating large slices with `dive` multiplies reflection work per element. Hot paths validating thousands of elements per request should benchmark; the map-dive-with-keys path is the most allocation-heavy in the suite[^5].
- **Unregistered tags panic.** A `validate:"require"` typo (missing the `d`) or a custom tag you forgot to register raises a runtime panic inside `Struct`, not a validation error. Cover your structs with a test that runs validation once so typos surface in CI, not production.
- **`required` semantics are subtle.** `required` fails on the type's zero value, so a legitimately-zero `int` or a `false` bool cannot be distinguished from "unset." Use pointers (`*int`) when zero is a valid value, and reach for `required_if` / `required_without` for conditional presence.
- **v10 opt-in flag foreshadows v11.** New instances should pass `validator.WithRequiredStructEnabled()`. Without it, `required` on a nested struct field behaves in the legacy way; the option enables the behavior slated to become default in v11+, so adopting it now avoids a silent semantics change on upgrade.
- **Error messages are your job.** Out of the box you get machine-readable `FieldError`s, not user-facing text. Budget for the translator wiring, or write a small mapper from `(Field, Tag)` to messages.
- **Gin coupling.** If you use Gin's `ShouldBindJSON`, you are already using this library; overriding its validator instance (to register custom tags or change the tag-name func) requires reaching into `binding.Validator`, not constructing your own[^2].
- **Not a sanitizer.** It validates, it does not mutate or coerce. Trimming, lowercasing, and defaulting are out of scope and must happen before or after.

## When to Use / When Not

**Use when:**
- You are validating request DTOs or config structs and want rules co-located with the fields.
- You are on Gin, or otherwise already depend on it transitively.
- You need broad format coverage (UUIDs, country codes, network addresses) without writing regexes.
- You want cross-field or nested-collection rules expressed declaratively.

**Avoid when:**
- You want validation errors caught at compile time — tags are strings, not types.
- Your rules are highly conditional or business-logic-heavy; programmatic builders read better than tag soup.
- You are validating loose `map[string]any` / dynamic payloads rather than typed structs.
- You need input transformation, not just yes/no validation.

## Alternatives

- go-ozzo/ozzo-validation — rules declared programmatically in Go, so typos are compile errors; better for conditional/complex logic, no struct-tag magic.
- asaskevich/govalidator — older, function- and tag-based; use for simple standalone format checks when you don't want the full engine.
- gookit/validate — supports structs and maps and includes filtering/sanitization; use when you need to validate dynamic map data or coerce values.
- invopop/validation — maintained fork in the ozzo lineage; use when you want the programmatic style with active upkeep.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1–v8 | 2015–2017 | Original tag-based engine under bluesuncorp/go-playground; API churned across majors. |
| v9 | 2017 | Package path moved to `go-playground/validator`; Gin migrated its default validator here[^2]. |
| v10 | 2019 | Current line. Import path `github.com/go-playground/validator/v10`; steady additions of format tags (crypto hashes, ISO codes, `jwt`, `semver`, `cron`) since. |
| — | 2023 | Open call for maintainers posted amid limited bandwidth[^3]. |
| v11 | planned | `WithRequiredStructEnabled` behavior to become default[^1]. |

## References

[^1]: Package README and usage docs — go-playground/validator. https://github.com/go-playground/validator
[^2]: Gin binding uses this package as its default struct validator. https://github.com/gin-gonic/gin/tree/master/binding
[^3]: "A Call for Maintainers" discussion #1330. https://github.com/go-playground/validator/discussions/1330
[^4]: Return-type rationale (typed-nil error pitfall), issue #134. https://github.com/go-playground/validator/issues/134
[^5]: Benchmark suite (Apple M3 Max, Go 1.23) in the project README. https://github.com/go-playground/validator#benchmarks

## Tags

go, golang, validation, struct-validation, struct-tags, reflection, gin, error-handling, i18n, data-validation, backend
