# DATA-DOG/go-sqlmock

> A mock implementation of Go's `database/sql/driver` for testing database interactions without a real database.

[GitHub repo](https://github.com/DATA-DOG/go-sqlmock) ·
License: BSD-3-Clause[^1]

## Overview

go-sqlmock is a test double for Go's standard `database/sql` package. It registers a fake SQL driver so that `sql.DB` sends queries into an in-memory expectation engine instead of a real server. You declare, up front, which statements the code under test should issue and what each should return; you then run the code and assert that every expectation was met. First published in 2014, it is one of the oldest and most-depended-on testing libraries in the Go ecosystem[^2].

The library is deliberately narrow. It does not parse SQL, does not maintain any table state, and does not understand your schema — it matches the query strings and arguments your code produces against the strings and arguments you predeclared, in order, and hands back whatever rows or errors you scripted. This makes it fast and dependency-free, but it also means a sqlmock test verifies *what SQL your code emits*, not *whether that SQL is correct*. That distinction is the central tension of the tool and the source of most complaints about it.

As of 2026 the project is explicitly in maintenance mode: the README states the library is "complete and stable," the last tagged release (v1.5.2) shipped in January 2024, and the author has an open request to transfer ownership to a motivated maintainer[^3]. It remains very widely used, so the low commit velocity reflects a finished design rather than abandonment — but new-feature contributions should not be expected.

## Getting Started

```bash
go get github.com/DATA-DOG/go-sqlmock
```

```go
package app

import (
	"database/sql"
	"testing"

	"github.com/DATA-DOG/go-sqlmock"
)

func TestRecordStats(t *testing.T) {
	db, mock, err := sqlmock.New() // returns a *sql.DB backed by the mock
	if err != nil {
		t.Fatalf("failed to open sqlmock: %s", err)
	}
	defer db.Close()

	mock.ExpectBegin()
	mock.ExpectExec("UPDATE products"). // matched as a regexp by default
		WillReturnResult(sqlmock.NewResult(1, 1))
	mock.ExpectExec("INSERT INTO product_viewers").
		WithArgs(2, 3).
		WillReturnResult(sqlmock.NewResult(1, 1))
	mock.ExpectCommit()

	if err := recordStats(db, 2, 3); err != nil {
		t.Errorf("unexpected error: %s", err)
	}
	// fails the test if any expectation was unmet or unexpected calls occurred
	if err := mock.ExpectationsWereMet(); err != nil {
		t.Errorf("unmet expectations: %s", err)
	}
}
```

The three-value `sqlmock.New()` constructor pings the connection on open so that `ExpectationsWereMet` is meaningful even if the code never touches the database. `NewWithDSN` lets you register a named mock and retrieve the same `*sql.DB` elsewhere via `sql.Open`, which is how you inject a mock into code that opens its own connection.

## Architecture / How It Works

sqlmock implements the `driver.Driver`, `driver.Conn`, `driver.Stmt`, `driver.Rows`, and (since Go 1.8) the context-aware `driver.*Context` interfaces. When you call `sqlmock.New()`, it registers a uniquely-named driver with `sql.Register` and opens a `*sql.DB` against it. Every method the standard library would call on a real driver — `Begin`, `Prepare`, `Exec`, `Query`, `Commit`, `Rollback` — instead consults an ordered list of `expectation` objects.

The matching model has two axes:

- **Query matching.** By default the expected string is compiled as a Go regular expression (`QueryMatcherRegexp`) and matched against the incoming query. This surprises many first-time users: `ExpectExec("INSERT INTO users (id) VALUES (?)")` fails because `(`, `)`, and `?` are regex metacharacters. You either escape them, match a distinctive substring, or switch to `QueryMatcherEqual` via `sqlmock.New(sqlmock.QueryMatcherOption(sqlmock.QueryMatcherEqual))` for exact case-sensitive comparison. Custom matchers implement the `QueryMatcher` interface, which is the extension point for AST-based validation[^4].
- **Argument matching.** Arguments are converted to `driver.Value` exactly as a real driver would, then compared. Non-comparable types (notably `time.Time`) are handled by implementing the `sqlmock.Argument` interface — the canonical `AnyTime{}` pattern asserts only the type — or by `sqlmock.AnyArg()` to skip a value entirely.

Expectations are **strictly ordered by default**: the code under test must issue statements in the exact sequence you declared. `mock.MatchExpectationsInOrder(false)` relaxes this to any-order matching, which is necessary for concurrent code or when statement order is not deterministic. Rows are built with `sqlmock.NewRows([]string{...}).AddRow(...)` or parsed from CSV via `FromCSVString`; a single `ExpectQuery` can return multiple row sets to model `Rows.NextResultSet`.

The package has no third-party dependencies — it is pure standard-library Go — which is a significant part of why it propagated so widely.

## Production Notes

- **It is a change-detector test.** Because matching happens on query strings, refactors that alter SQL text without changing behavior (reordering columns, adding whitespace, switching a JOIN style) break tests even though the code is still correct. Conversely, a query that is syntactically valid to sqlmock but rejected by the real database passes. Teams that lean heavily on sqlmock often discover their test suite is green while production queries fail. Treat sqlmock as a unit-level guard on *which* statements run, and cover real SQL correctness with integration tests against an actual engine.
- **The default regexp matcher is the top footgun.** Silent partial matches (the expected pattern is a substring regex, so `"SELECT"` matches almost anything) and unescaped metacharacters cause both false passes and confusing failures. Many teams standardize on `QueryMatcherEqual` project-wide to make matches explicit.
- **Ordering strictness bites concurrent code.** Connection-pool nondeterminism and goroutine interleaving will make in-order expectations flaky; remember to disable ordering when the code under test is not strictly sequential.
- **ORM coupling.** With GORM, sqlx, ent, and similar libraries, the SQL that reaches the driver is generated and may include quoting, backticks, `RETURNING` clauses, or driver-specific syntax you did not write. Expectations must match the ORM's output, not your mental model of the query, which makes sqlmock tests brittle across ORM upgrades. Some ecosystems (pgx) are not covered at all — see pashagolub/pgxmock below.
- **Maintenance risk.** With the author seeking a new owner and no release since early 2024, do not build workflows that depend on upstream fixes or new driver-interface support landing quickly[^3]. The design is stable, but Go driver-interface additions in future toolchain versions may not be tracked promptly.
- **Licensing metadata.** GitHub reports the license as unrecognized (`NOASSERTION`) because the repository ships the BSD text inline rather than in a detected `LICENSE` file; the README states the three-clause BSD license[^1]. Verify against your compliance tooling if automated license scanning matters to you.

## When to Use / When Not

**Use when:**
- You want fast, hermetic unit tests that assert your data-access layer issues the expected statements, transactions, and arguments.
- You need to simulate error paths (connection failures, constraint violations, rollback behavior) that are hard to trigger against a real database.
- You cannot or do not want to stand up a database in the test environment, and you accept that you are testing SQL emission, not SQL correctness.

**Avoid when:**
- You need confidence that your queries actually run against your target engine — use a real database via containers or transactional isolation instead.
- Your queries are generated by an ORM and change shape across versions; the string coupling makes tests high-maintenance.
- You are on pgx (not `database/sql`) — sqlmock mocks the `database/sql/driver` layer and does not cover the pgx-native interface.

## Alternatives

- DATA-DOG/go-txdb — same author; runs tests against a **real** database but isolates each test inside a single transaction that is rolled back. Use when you want real SQL execution with fast cleanup.
- testcontainers/testcontainers-go — spins up a throwaway database in Docker per test suite. Use when correctness against the actual engine matters more than speed.
- ory/dockertest — lighter-weight Docker orchestration for ephemeral test databases. Use when you want real databases without the testcontainers dependency surface.
- pashagolub/pgxmock — a sqlmock-style mock for the pgx driver interface. Use when your code targets pgx directly rather than `database/sql`.
- Repository/interface mocking (hand-written or gomock) — mock your own data-access interface above the SQL layer. Use when you want to test business logic and treat the database boundary as an opaque contract.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2014 | Initial release; mock of `database/sql/driver`[^2]. |
| v1.0.0 | 2015-08-27 | v1 API; concurrency support, known issues resolved. |
| v1.1.0 | 2016-02-23 | `AnyArg()` argument matcher; stricter `driver.Value` conversion. |
| v1.2.0 | 2017-02-21 | Go 1.8 features; `Rows` changed from interface to struct (breaking). |
| v1.3.0 | 2017-10-04 | Prepared-statement close expectations. |
| v1.3.2 | 2019-02-12 | `go.mod` added; `gopkg.in` references dropped. |
| v1.4.0 | 2020-01-06 | `QueryMatcher` interface; regexp vs. equal matching. |
| v1.5.0 | 2020-08-14 | Additional context/driver-method coverage. |
| v1.5.2 | 2024-01-06 | Latest release; project declared complete and stable[^3]. |

## References

[^1]: License stated as the three-clause BSD license in the project README; GitHub's classifier reports `NOASSERTION`. https://github.com/DATA-DOG/go-sqlmock#license
[^2]: Repository created 2014-02-07. https://github.com/DATA-DOG/go-sqlmock
[^3]: "Looking for maintainers" section and maintenance status, README (issue #230). https://github.com/DATA-DOG/go-sqlmock/issues/230
[^4]: `QueryMatcher` customization, README. https://github.com/DATA-DOG/go-sqlmock#customize-sql-query-matching

## Tags

go, golang, testing, database, sql, mock, test-double, database-sql, tdd, unit-testing
