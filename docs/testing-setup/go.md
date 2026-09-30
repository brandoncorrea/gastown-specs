# Testing Setup — Go

Go-specific testing conventions for this project. For general testing philosophy, see
`/docs/testing-standards.md`.

## Framework

- **Runner:** the standard `testing` package and `go test`. No BDD frameworks (Ginkgo,
  GoConvey) — they replace Go's test model with their own and hide it from the tooling.
- **Assertions:** `github.com/stretchr/testify`. Lean on it as much as possible; drop
  to hand-written checks only when testify can't express the assertion.
- **HTTP:** `net/http/httptest` from the standard library.
- **Time:** `testing/synctest` from the standard library.
- **Mutation testing:** `gremlins`, configured in `.gremlins.yaml` (or the project's
  equivalent).

## File Organization

Go decides most of this. Tests sit **beside** the code they test, in files ending
`_test.go`. There is no separate test tree.

```
internal/
  orders/
    create.go
    create_test.go            ← package orders_test
    cancel.go
    cancel_test.go
    order.go                  ← no order_test.go: covered through create and cancel
  payment/
    client.go
    client_test.go
    paymenttest/              ← test support for package payment
      paymenttest.go
```

Not every source file needs a test file — see "Implicit Coverage Is Real Coverage" in
`/docs/testing-standards.md`.

If a test file grows too large, split it by behavior, not by number:
`create_validation_test.go`, `create_pricing_test.go`, `create_conflict_test.go`.

### Test Packages: Black-Box by Default

A `_test.go` file may declare either the package itself (`package orders`) or the
external test package (`package orders_test`). **Use the external `_test`
package by default.** It can only see exported identifiers, so the compiler enforces
"test behavior, not implementation" for you.

```go
package orders_test

import (
	"testing"

	"github.com/stretchr/testify/require"

	"example.com/shop/internal/orders"
)
```

White-box tests (`package orders`) are allowed when there is a reason you can
state in one sentence — typically an unexported pure function with enough edge cases to
deserve its own table. Before reaching for one, ask whether that function wants to be
its own package instead.

### When a Black-Box Test Can't Reach Something

Moving a test to the external package will sometimes leave it unable to reach something
it used to touch. Work down this list:

1. **Assert the behavior, not the internal.** Often the test never needed the
   identifier. A test that used `maxErrorDetail` to compute its input can state the
   contract directly — send 1024 bytes, expect exactly 512 back. The literal *is* the
   specification; if someone changes the limit, the test should fail. This is the
   stricter test, and it costs no export.
2. **Just export it.** Everything under `internal/` and in `package main` can only be
   imported by this module, so an exported name there is a promise to nobody but us.
   Export the field, constant, or function, get the initialisms right (`HTTP`, not
   `Http`), and move on. Sometimes this improves the design: a `ShutdownTimeout` field
   is a legitimate configuration knob, not just a test hook.
3. **Stop and think when the package is importable.** For a package other modules can
   import (anything outside `internal/`), an export *is* a promise to strangers.
   Prefer option 1; if it truly can't work, a white-box test for that one case is
   better than widening a public API.

**We don't use `export_test.go`.** The standard library uses that idiom — a file in the
internal package, compiled only under `go test`, that re-exports internals — because
every name it exports is a permanent public promise. Here it is ceremony: the test calls
`srv.HTTPServer()`, the reader opens `server.go`, and the method isn't there. That
indirection costs more than the export it avoids.

The one thing to watch when exporting a field: an exported *mutable* field lets any
package in the module change it. If an invariant depends on it staying put (the read
timeouts on the HTTP server), say so in a comment on the field.

## Test Naming

```go
func TestRejectsARefundPastTheRefundWindow(t *testing.T) {
	// Arrange
	// Act
	// Assert
}

func TestResolvesLogLevelFromEnvironment(t *testing.T) {
	cases := []struct {
		name  string
		value string
		want  slog.Level
	}{
		{name: "defaults to info when unset", value: "", want: slog.LevelInfo},
		{name: "accepts lowercase names", value: "debug", want: slog.LevelDebug},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			t.Setenv("LOG_LEVEL", tc.value)
			require.Equal(t, tc.want, logging.ResolveLevel())
		})
	}
}
```

Top-level names are the behavior in `MixedCaps`. Subtest names are plain sentences.
Table-driven tests follow the rules in `/docs/testing-standards.md`: one behavior per table,
every case named, no branching in the loop body.

## Setup and Teardown

Go has no hooks. Use these instead:

| Need | Go |
|------|-----|
| Shared arrange | A builder function: `newHandler(t)` returning what the test needs |
| Teardown | `t.Cleanup(func() { ... })`, registered inside the builder |
| Helper that asserts | First line is `t.Helper()` so failures point at the test, not the helper |
| Temp files | `t.TempDir()` — removed automatically |
| Environment variables | `t.Setenv(key, value)` — restored automatically |
| A context | `t.Context()` — cancelled when the test ends |
| Process-wide setup | `TestMain`, and only for things that are truly process-wide |

```go
func newHandler(t *testing.T) (*orders.Handler, *orders.InMemoryStore) {
	t.Helper()
	placed := orders.NewInMemoryStore()
	gateway := payment.NewInMemoryGateway()
	return orders.New(placed, gateway), placed
}
```

Builders return fresh values. Nothing is shared between tests, so nothing needs
resetting. Production types are built with their `New` constructor, in tests exactly as
in `main`.

### Naming Locals in Black-Box Tests

In an external test package every reference is qualified, so the obvious local name is
usually taken by the package: `server := server.Listen(...)` compiles, but it shadows
the package for the rest of the function and the next `server.Something` is a
confusing error.

Name the local for the **role it plays in the test**. When the thing already *is* a
domain term, keep the term and qualify it rather than reaching for a synonym:

| Instead of | Prefer |
|---|---|
| `server := server.Listen(...)` | `listening`, or the conventional `srv` |
| `db := orders.NewInMemoryStore()` | `placed`, `orderStore` |
| `payment := payment.NewInMemoryGateway()` | `gateway`, `declinedGateway` |
| `customer := customer.New(...)` | `buyer`, `repeatCustomer` |

When the domain already has a name for the thing, keep it and qualify it rather than
reaching for a clever synonym — and a synonym is actively wrong when it is itself a
different domain term. See "Reserved Domain Vocabulary" in
`/docs/clean-code-standards.md`.

Short conventional names (`srv`, `req`, `rec`, `ctx`) are fine — name length scales with
scope, and a test body is a small scope. Invented abbreviations (`mem`, `hdl`, `fp`)
are not: if a reader has to guess what it abbreviates, spell out the role instead.

### `testify/suite`

Permitted, not the default. `suite` gives you `SetupTest`/`TearDownTest`, which is the
closest thing Go has to `before`/`after`. It costs you: tests become methods,
every assertion becomes `s.Require().Equal(...)`, `go test -run` no longer selects a
single test (you need `-testify.m`), suites can't use `t.Parallel`, and suite fields
are shared mutable state unless `SetupTest` rebuilds every one of them.

Use a suite when **three or more tests in a file share non-trivial arrange *and*
teardown**, and the builder-function version is visibly noisier. When you do:
- Rebuild all state in `SetupTest`. Never hold mutable state from `SetupSuite`.
- One suite per file, named for the unit under test.

## Assertions

Testify first. `require` stops the test at the first failure; `assert` records the
failure and keeps going.

- **Use `require` by default.** A failed precondition followed by twenty cascading
  nil-pointer failures helps nobody.
- **Use `assert` only off the test goroutine** — inside an `http.HandlerFunc` run by a
  test server, or any goroutine the test started. `require` calls `t.FailNow`, which
  is only legal on the goroutine running the test.
- **Argument order is `(t, expected, actual)`.**
- **Reach for the specific assertion.** Never `require.True(t, a == b)`.

| Pattern | Testify |
|---------|---------|
| Equality | `require.Equal(t, expected, actual)` |
| Equality across numeric / named types | `require.EqualValues(t, 1, order.Version)` |
| Error expected / not expected | `require.Error(t, err)` / `require.NoError(t, err)` |
| A specific error | `require.ErrorIs(t, err, orders.ErrNotFound)` |
| An error of a type | `require.ErrorAs(t, err, &conflict)` |
| Collection size / emptiness | `require.Len(t, items, 1)` / `require.Empty(t, items)` |
| Membership | `require.Contains(t, orderIDs, orderID)` |
| Same elements, any order | `require.ElementsMatch(t, expected, actual)` |
| JSON, ignoring formatting and key order | `require.JSONEq(t, expected, actual)` |
| Times within a tolerance | `require.WithinDuration(t, expected, actual, time.Second)` |
| Something becomes true | `require.Eventually(t, condition, waitFor, tick)` |
| A panic | `require.Panics(t, func() { ... })` |

When the same multi-step assertion appears three times, extract a helper that takes
`t`, calls `t.Helper()`, and reads as one assertion — `RequireJSON(t, recorder, want)`.

Drop to the standard library only when testify can't say it — for example
`t.Skip`, `t.Fatalf` inside `TestMain`-adjacent plumbing, or fuzz targets.

## Test Doubles

In order of preference:

1. **A real in-memory implementation.** An in-memory store or gateway that satisfies
   the same interface as the real one is production code — ideally `main` can wire it
   in as a runtime mode for local development — and tests should use it the same way.
   Assert on the resulting *state*, not on which methods were called.
2. **A typed function-field stub**, when you need to inject a failure or observe a
   call that the in-memory implementation can't produce:

   ```go
   type Stub struct {
   	ChargeFn func(ctx context.Context, amount payment.Amount) (payment.Receipt, error)
   }

   func (s Stub) Charge(ctx context.Context, amount payment.Amount) (payment.Receipt, error) {
   	return s.ChargeFn(ctx, amount)
   }

   // In the test — terse, typed, and a rename breaks the build instead of the test run
   unavailableGateway := paymenttest.Stub{
   	ChargeFn: func(context.Context, payment.Amount) (payment.Receipt, error) {
   		return payment.Receipt{}, errUnavailable
   	},
   }
   ```
3. **`testify/mock`**, as a last resort, when you must verify a *sequence* of
   interactions that option 2 makes awkward. `mock.On("MethodName", ...)` is
   string-typed: renames break it silently, and neither `gopls` nor an agent can trace
   it. If you reach for it, say why in the test.

Never substitute the code under test.

### Where Test Support Lives

Follow the standard library: test support for package `foo` lives in a subpackage named
`footest` — like `net/http/httptest`, `testing/fstest`, and `log/slog/slogtest`.
The kinds of thing they typically hold:

| Package | Holds |
|---|---|
| `internal/payment/paymenttest` | `NewClient`: a `payment.Client` wired to a test server; `Stub`, a function-field stub of the gateway |
| `internal/logging/logtest` | A logger with a recorder for asserting on log entries |
| `internal/ordertest` | Named domain fixtures (`NewOrder`, `NewCustomer`) used across many packages |
| `internal/apitest` | HTTP helpers: `RequireJSON`, a failing transport, a must-not-be-called handler, route assertions |

Keep a table like this one in the project's own copy of this doc, listing the support
packages that actually exist, so agents find them before writing new ones.

- Anything only tests import does **not** go in the production package, and does not
  go in a catch-all `testutil` or `helpers`
- Support for one package goes beside it as `footest`. Support that serves many
  packages gets a package named for the *subject* it covers (`ordertest`, `apitest`),
  still ending in `test`
- Fixtures are functions returning fresh values, never package-level variables
- Every helper that takes `t` calls `t.Helper()` first

The `test` suffix is load-bearing: the mutation-testing config excludes test support by
that suffix, so a support package named anything else will be mutated and report
meaningless survivors.

### HTTP

- **Handlers** — build a request with `httptest.NewRequest`, record with
  `httptest.NewRecorder`, call the handler. Set path values with `r.SetPathValue`.
- **Outbound clients** — point the client at an `httptest.NewServer` whose handler
  plays the remote service. Close it with `t.Cleanup(server.Close)`. Inside that
  handler, use `assert`, not `require`.
- **Transport failures** — an `http.RoundTripper` that returns an error.
- **Never touch the real network** from `go test`.

### Time

Never assert against `time.Now()`, and never `time.Sleep`.

- **`testing/synctest`** runs a test inside a bubble with a fake clock that advances
  only when every goroutine is blocked. Timeouts, tickers, and `time.Now()` become
  deterministic and instant, with no injection required:

  ```go
  func TestTokenIsRefreshedAfterItExpires(t *testing.T) {
  	synctest.Test(t, func(t *testing.T) {
  		source := newTokenSource(t)
  		first := mustToken(t, source)

  		time.Sleep(tokenLifetime + time.Second) // instant inside the bubble

  		require.NotEqual(t, first, mustToken(t, source))
  	})
  }
  ```
- Where a bubble doesn't fit (real I/O is involved), inject the clock as a
  `func() time.Time` field.

## Testing HTTP Services

See "Testing HTTP Services" in `/docs/testing-standards.md` for the rationale.

**Logic tests** call the handler method directly:

```go
func TestCreatesAnOrderForTheAuthenticatedCustomer(t *testing.T) {
	handler, placed := newHandler(t)
	recorder := httptest.NewRecorder()

	handler.CreateOrder(recorder, createOrderRequest(t, ordertest.NewOrderBody()))

	require.Equal(t, http.StatusCreated, recorder.Code)
	require.Len(t, placed.All(), 1)
}
```

**Route tests** assert on the wiring and nothing else — which pattern a method and path
resolve to, and which handler method is registered there. No handler runs, so a route
test never touches a status code or a body that belongs to the handler's contract:

```go
func TestRoutes(t *testing.T) {
	cases := []struct {
		name    string
		method  string
		path    string
		pattern string
	}{
		{name: "creates orders", method: http.MethodPost, path: "/orders", pattern: "POST /orders"},
		{name: "cancels one order", method: http.MethodDelete, path: "/orders/ORDER_ID", pattern: "DELETE /orders/{id}"},
	}
	mux := http.NewServeMux()
	orders.New(orders.NewInMemoryStore(), paymenttest.Stub{}).RegisterRoutes(mux)
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			_, pattern := mux.Handler(httptest.NewRequest(tc.method, tc.path, nil))
			require.Equal(t, tc.pattern, pattern)
		})
	}
}
```

The test calls the package's real `RegisterRoutes` and asks the mux what each request
resolves to with `ServeMux.Handler` — nothing is served. To also pin *which* handler
method sits behind a pattern, put a `RequireRouteRegistration`-style helper in a shared
test-support package (comparing function pointers there is an acceptable use of
`reflect`). Each handler package owns its route strings; a top-level router only loops
over `RegisterRoutes`, and its test checks exactly that with a tiny registrar of its
own. Middleware (auth especially) gets its own test proving an unauthenticated request
never reaches the handler.

## Beyond Unit Tests

- **Fuzzing** — anything that parses or validates untrusted input gets a fuzz target
  (`func FuzzParseAddress(f *testing.F)`). Seed it with the named fixtures. A crash the
  fuzzer finds becomes a regular regression test.
- **Mutation testing** — `gremlins unleash` mutates the production code and reruns the
  tests. Surviving mutants in business logic are missing tests. Test support and
  generated code are excluded in the mutation config; keep that list honest when files
  move.
- **Integration tests** — tests that need a real database or other infrastructure are
  kept out of the default `go test ./...` run, behind a build tag
  (`//go:build integration`) or a `testing.Short()` / environment-variable skip.
- **Acceptance / end-to-end** — if the project has a suite that runs against a live
  instance of the service, it is the acceptance layer. Unit tests don't replace it, and
  it doesn't replace unit tests. If it needs external infrastructure or takes minutes,
  **the maintainer runs it** — agents don't; if a change could affect it, say so at
  handoff.

## Running Tests

Go tests run by **package**, not by file, and results are cached per package — an
unchanged package reports `(cached)` and costs nothing. Running `./...` is cheap.

```bash
# Entire suite, as CI runs it (or the project's `make test`)
go test -race ./...

# One package
go test ./internal/orders

# One test (regex on the name); add -v for subtest detail
go test ./internal/orders -run 'TestCreatesAnOrderForTheAuthenticatedCustomer'

# One subtest (spaces in subtest names become underscores)
go test ./internal/orders -run 'TestRoutes/creates_orders'

# Bypass the cache (flaky-test hunting)
go test -count=1 ./internal/server

# Failures only, as JSON
go test -json ./... | jq -r 'select(.Action=="fail") | "\(.Package) \(.Test // "")"'

# Coverage for one package
go test -cover ./internal/orders

# Fuzz one target for 30 seconds
go test ./internal/orders -run '^$' -fuzz FuzzParseAddress -fuzztime 30s

# Mutation testing
gremlins unleash
```

There is no watch mode in the Go toolchain; with per-package caching, rerunning the
command is the watch mode.

CI should run `go vet ./...`, `go build ./...`, and `go test -race ./...` on every push
and pull request.
