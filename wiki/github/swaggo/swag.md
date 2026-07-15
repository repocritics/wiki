# swaggo/swag

> Generates Swagger 2.0 API docs for Go by parsing declarative comments above your handlers.

[GitHub repo](https://github.com/swaggo/swag) ·
[License: MIT](https://github.com/swaggo/swag/blob/master/LICENSE)

## Overview

Swag is a Go code-generation tool that reads specially-formatted comments (`// @Summary`, `// @Param`, `// @Router`, …) above your HTTP handlers and struct definitions, walks the source with Go's AST parser, and emits an OpenAPI/Swagger **2.0** specification as `swagger.json`, `swagger.yaml`, and a `docs/docs.go` file that embeds the spec into your binary[^1]. Paired with a framework adapter (`gin-swagger`, `echo-swagger`, `http-swagger`), it serves an interactive Swagger UI at a route in your app. It has been the default "how do I get API docs in Go" answer since roughly 2018, and the surrounding `swaggo/*` adapters make it a small ecosystem rather than a single tool.

The defining tension is the version number: swag generates **Swagger 2.0**, a specification frozen in 2014 and superseded by OpenAPI 3.0 (2017) and 3.1 (2021). A rewrite that targets OpenAPI 3 has existed on the `v2` branch since early 2023, but as of 2026 it is still in release-candidate status (`v2.0.0-rc5`, January 2026) after three years[^2]. Teams that need OpenAPI 3.0/3.1 output — for tooling, code generators, or newer Swagger UI features — cannot get there with stable swag, and this is the single most consequential fact to know before adopting it.

The second thing to internalize is that swag is **annotation-driven, not type-driven**. The spec is only as correct as the comments; the Go compiler does not check them. Rename a field, change a status code, or return a different struct and the docs silently drift until someone re-runs `swag init`. This is the opposite tradeoff from spec-first tools (go-swagger, oapi-codegen) that treat the spec as the source of truth and generate code from it.

## Getting Started

```sh
go install github.com/swaggo/swag/cmd/swag@latest   # requires Go 1.19+
```

Annotate a handler, then generate:

```go
// ShowAccount godoc
// @Summary      Show an account
// @Description  get an account by ID
// @Tags         accounts
// @Produce      json
// @Param        id   path      int  true  "Account ID"
// @Success      200  {object}  model.Account
// @Failure      404  {object}  httputil.HTTPError
// @Router       /accounts/{id} [get]
func (c *Controller) ShowAccount(ctx *gin.Context) { /* ... */ }
```

```sh
swag init                 # scans ./, writes ./docs (docs.go, swagger.json, swagger.yaml)
swag init -g api/main.go  # if your @title/@version block isn't in main.go
swag fmt                  # gofmt-style alignment for the comment blocks
```

Then import the generated package for its `init()` side effect and wire the UI:

```go
import _ "your-module/docs"
// r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
```

## Architecture / How It Works

Swag is a static analyzer. `swag init` parses your packages with `go/ast` and `go/parser`, collects the comment groups attached to functions and types, and runs them through a hand-written parser for the `@`-annotation grammar. It resolves referenced types (`{object} model.Account`) by loading and inspecting the corresponding struct declarations, reading Go struct tags (`json:"..."`, `example:"..."`, `binding:"required"`, plus swag-specific `swaggertype`, `swaggerignore`) to build the JSON schema definitions. The output is an in-memory Swagger 2.0 document serialized to JSON/YAML, plus a Go file that hard-codes that JSON as a string and registers it with the `swag` runtime registry so framework adapters can retrieve it by instance name.

Because it is comment-based over real source, several things follow directly from the design:

- **Type resolution crosses packages but not the module boundary by default.** Types from your own module resolve automatically. Types from dependencies (a `time.Time`, a `uuid.UUID`, a struct from a third-party lib) require `--parseDependency`/`--parseInternal`, or a `swaggertype`/`--overridesFile` mapping, or they surface as empty/incorrect schemas.
- **Generics support is newer and partial.** Instantiated generic types (`Response[User]`) are supported from the v1.16 line (2023) using a `Response-User` naming convention, but coverage is narrower than concrete types and remains a common source of bug reports.
- **The generated `docs.go` is a build artifact you commit.** It changes on every annotation edit, so it shows up in diffs and must be regenerated in CI or via a `go:generate` directive to stay in sync.
- **Adapters are separate repos.** `swaggo/swag` produces the spec; `swaggo/gin-swagger`, `swaggo/echo-swagger`, `swaggo/http-swagger` (gorilla/mux, chi, net/http) and community adapters (fiber, hertz, atreugo) serve it. Version skew between swag and an adapter's embedded `swaggo/files` is a recurring integration snag.

## Production Notes

- **Docs drift is the number-one footgun.** Nothing fails the build when a comment lies. The mitigation is to make `swag init` non-optional: run it in CI and fail if `git diff --exit-code docs/` is dirty, so stale docs cannot merge.
- **Parsing large trees is slow and memory-hungry.** `--parseDependency --parseInternal` on a big module can take tens of seconds and a lot of RAM because swag loads dependency source to resolve types. Use `--dir` to scope, `--exclude` to prune, and `-q` to quiet output in CI. `--parseDepth` caps recursion.
- **Third-party and standard-library types need overrides.** `time.Time`, `decimal.Decimal`, `json.RawMessage`, `uuid.UUID`, and `sql.Null*` commonly render wrong. Fix them once with a `.swaggo` overrides file or field-level `swaggertype:"string"` / `swaggertype:"primitive,integer"` tags rather than fighting per-endpoint.
- **You are locked to Swagger 2.0 semantics.** No `oneOf`/`anyOf`, no true nullable, single request/response content type per operation, `definitions` instead of `components/schemas`. Downstream OpenAPI-3-only codegen or linters will reject the output; you convert with `swagger2openapi` or accept the ceiling.
- **The v2/OpenAPI-3 branch is not a drop-in upgrade.** It is a rewrite with different behavior and long-lived RC status; do not plan a production migration around an unreleased RC. Treat "we'll switch to v2 later" as a risk, not a plan.
- **Comment formatting is finicky.** `swag fmt` requires a leading standard doc comment before the annotation block or it will not indent correctly, and misaligned/typo'd annotations are silently ignored rather than erroring — so a missing `@Router` just drops the endpoint from the spec with no diagnostic.

## When to Use / When Not

**Use when:**
- You have an existing Go HTTP service (especially Gin/Echo) and want docs + Swagger UI with minimal restructuring.
- Swagger 2.0 output is acceptable for your consumers and tooling.
- You prefer keeping documentation next to the handler code (code-first) over maintaining a separate spec file.

**Avoid when:**
- You need OpenAPI 3.0 or 3.1 output — reach for oapi-codegen, ogen, or huma instead.
- You want the spec to be the contract and the code to be generated/validated against it (spec-first) — swag is the inverse.
- You want compile-time guarantees that docs match handlers; annotations are unchecked strings.

## Alternatives

- go-swagger/go-swagger — mature Swagger 2.0 toolkit, spec-first: generates server/client from a spec and validates it. Use when you want the spec as source of truth and richer 2.0 tooling, and can accept a heavier workflow.
- oapi-codegen/oapi-codegen — spec-first OpenAPI 3.0 → Go types/servers/clients. Use when you're on OpenAPI 3 and want the contract to drive the code.
- ogen-go/ogen — OpenAPI 3.0 code generator focused on performance and a fully typed, reflection-free server. Use when you want OpenAPI 3 with a strict, fast generated layer.
- danielgtaylor/huma — code-first framework that emits OpenAPI 3.1 from Go types and validates requests at runtime. Use when you want swag's code-first ergonomics but on modern OpenAPI without a separate generate step.
- goadesign/goa — design-first DSL that generates code, docs, and OpenAPI. Use when you want a single Go DSL as the contract for larger service suites.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-06 | Repository created; comment-driven Swagger 2.0 generation for Go[^1]. |
| v1.6.0 | 2019-07 | Established 1.x line; broader annotation coverage. |
| v1.7.0 | 2020-12 | Continued 1.7 series of parser/type-resolution fixes. |
| v1.8.0 | 2022-02 | 1.8 series; ongoing framework-adapter and type-mapping work. |
| v2.0.0-beta / rc1 | 2023-04 | First public OpenAPI-3-targeting rewrite on the v2 branch[^2]. |
| v1.16 line | 2023 | Go generics support (`Type-Param` instantiation naming). |
| v1.16.4 | 2024-10 | Stable 1.16 maintenance release. |
| v1.16.6 | 2025-07 | Latest stable 1.x at time of writing. |
| v2.0.0-rc5 | 2026-01 | OpenAPI 3 rewrite still in release-candidate after ~3 years[^2]. |

As of the last fetch the repo has ~13.0k stars, ~1.5k forks, and ~460 open issues — actively maintained on the 1.x line (last push July 2026), with the OpenAPI-3 v2 effort the dominant open question hanging over adoption.

## References

[^1]: swaggo/swag README — "Swag converts Go annotations to Swagger Documentation 2.0." https://github.com/swaggo/swag
[^2]: swaggo/swag releases — `v2.0.0-beta`/`rc1` (April 2023) through `v2.0.0-rc5` (January 2026), the OpenAPI 3 branch. https://github.com/swaggo/swag/releases

## Tags

go, golang, swagger, swagger2, openapi, api-documentation, code-generation, annotations, rest-api, gin, developer-tools
