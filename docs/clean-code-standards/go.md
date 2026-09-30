# Clean Code Standards — Go

Clean code is a UX problem. Our users are other developers, coding agents, and our
future selves — we're creating a user experience for people who read our code.

Like art, code quality is almost always subjective, but you can tell good code from bad
code because the good code follows rules. This document defines those rules. Any of
them can be broken — but only with reason and intention. Breaking a rule because you
thought about it and decided the code is clearer without it is clean code. Breaking a
rule because you didn't know it existed is not.

## Go Conventions Win

When a Go convention conflicts with a general clean-code heuristic, the Go convention
wins. These conventions aren't just style — they are what `gofmt`, `go vet`, `gopls`,
and every Go developer expect. A name that's "technically more readable" by general
rules but fights the language is not clean — it's wrong.

- **`gofmt -s` is the final word on layout.** Indentation, alignment, brace placement,
  trailing commas in multi-line literals, import sorting. Never argue with it, never
  hand-format around it. Run `gofmt -s -w .` (or the project's format target).
- **`MixedCaps` for everything.** No underscores, no `SCREAMING_SNAKE` constants.
  `maxLoginAttempts`, not `MAX_LOGIN_ATTEMPTS`.
- **Initialisms keep one case.** `userID`, `baseURL`, `httpClient`, `ParseJSON` —
  not `userId`, `BaseUrl`.
- **`ctx context.Context` is the first parameter; `error` is the last return value.**
- **HTTP handlers have the signature `func(w http.ResponseWriter, r *http.Request)`.**
  That shape is fixed by `net/http`; don't wrap it in something cleverer.
- **Latest Go, latest idioms.** The project tracks the latest Go release, so write for
  it. When the standard library has a newer way, use it: `slices`, `maps`, `min`/`max`,
  `for i := range n`, iterators, `errors.AsType`, `sync.WaitGroup.Go`, `t.Context()`.
  `go fix ./...` applies most of these modernizations mechanically; run it after
  upgrading Go.

### Generated Code Is Exempt

Any file carrying Go's `// Code generated ... DO NOT EDIT.` header — from an OpenAPI
spec, protobuf, `sqlc`, `stringer`, or anything else — is exempt from every rule in this
document, and its names are used **verbatim** (`UserId`, `ApiUrl`,
`OrderStatus_Pending`) even when they break the initialism rule. Never hand-edit
generated code; change its source and regenerate. Never "fix" its names at the call
site with aliases.

## Naming

Names are the most important tool for readability. Invest time in them.

- **Name length scales with scope.** A loop index is `i`. A value used across forty
  lines is `remainingAttempts`. The further a name travels from its declaration, the
  more it must explain itself.
- **Go's short names are idiom, not abbreviation.** `ctx`, `err`, `ok`, `w`, `r`, `t`,
  `i`, and one- or two-letter method receivers (`h *Handler`) are universally understood.
  Use them. Invented abbreviations (`usr`, `mgr`, `fpln`) are still not okay.
- **Variables** — describe what they hold, not their type. `remainingAttempts`, not
  `num` or `intVal`.
- **Functions** — describe what they do with a verb. `calculateDiscount`, not
  `discount` or `doStuff`.
- **Booleans** — read as a yes/no question. `isExpired`, `hasPermission`,
  `shouldRetry`.
- **Constants** — describe the meaning, not the value. `maxLoginAttempts`, not `three`.
  Prefer typed constants (`const refundWindow = 30 * 24 * time.Hour`).
- **Avoid generic names** — `data`, `info`, `item`, `result`, `temp`, `value` almost
  always have a better name.
- **Don't stutter.** The package name is part of every exported name. `user.Service`,
  not `user.UserService`. `db.InMemory`, not `db.InMemoryDB`.
- **Getters don't say `Get`** when they merely return a field: `order.Total()`, not
  `order.GetTotal()`. `Get` is fine for operations that fetch (`store.GetOrder`) and
  wherever it mirrors an API's operation names.

If you need a comment to explain what a variable or function does, the name is wrong.
Rename it.

### Reserved Domain Vocabulary

Every domain has ordinary-sounding words with a precise meaning — often fixed by a
standard, a regulation, or the business itself. In this codebase — names, comments, log
messages, docs — **they mean that and nothing else.** Using one loosely doesn't just
read badly; it tells the next reader something false about the system.

List them here. The table starts with an example row; replace it with your domain's
terms:

| Word | Means only | Not |
|---|---|---|
| **settlement** *(example, payments)* | Funds actually moving between institutions | Authorizing or capturing a charge |

The rule when naming something the domain already named: **use the domain's term and
qualify it** (`failedSettlement`, `pendingRefund`, `staleVersion`). Don't invent a
synonym, and never borrow a different domain term as a metaphor.

This list grows. If you catch a word being used in two senses, add it here.

### Exporting

Unexported is the default. Capitalize a name only when another package needs it.
Over-exporting is a smell: every exported identifier is a promise, and it hides which
parts of a package are the contract and which are plumbing.

A package's own black-box tests count as "another package." Under `internal/` and in
`package main`, exporting something so its tests can reach it is fine — see "When a
Black-Box Test Can't Reach Something" in `/docs/testing-setup.md`. In a
package other modules can import, it is not.

## Functions

### Do One Thing

A function does one thing if you cannot meaningfully extract another function from it.
If you can describe it only with "and" or "then," split it.

### Keep Them Short

Aim for ~20 lines. This isn't a hard ceiling, but past 20, look for extraction
opportunities. Past 40, it almost certainly does more than one thing.

`if err != nil { return ... }` guard clauses don't count toward the total. They are
Go's cost of doing business, not logic.

### Limit Parameters

- 0-2 parameters: ideal
- 3 parameters: acceptable, consider an options struct
- 4+ parameters: refactor into an options struct or split the function

`ctx context.Context` and the fixed `(w, r)` handler pair don't count toward the limit.

```go
// Bad — what do these arguments mean at the call site?
createWebhook(ctx, "https://hooks.example", 5, true, 30)

// Good — a struct is self-documenting
createWebhook(ctx, WebhookOptions{
	CallbackURL:    "https://hooks.example",
	MaxRetries:     5,
	SignPayloads:   true,
	DeliveryWindow: 30 * time.Minute,
})
```

### Phantom Parameters

If a parameter always receives the same value at every call site, it's not a real
parameter — it's a phantom. Hard-code it into the function and remove it from the
signature. The parameter can always be reintroduced later if requirements actually
demand it. Until then, it's noise that every caller has to know about for no reason.

```go
// Every caller passes time.RFC3339Nano — it's not a real choice
parseTime(start.Value, time.RFC3339Nano)
parseTime(end.Value, time.RFC3339Nano)

// Just hard-code it
func parseTime(value string) (time.Time, error) {
	return time.Parse(time.RFC3339Nano, value)
}
```

### No Flag Arguments

A boolean parameter that switches a function between two behaviors means the function
does two things. Split it.

```go
// Bad — what does `true` mean here?
saveOrder(ctx, order, true)

// Good — two functions with clear names
createOrder(ctx, order)
updateOrder(ctx, order)
```

### Minimize Nesting

Maximum 2 levels of indentation inside a function. Keep the happy path on the left
edge; deeper nesting obscures logic.

**Techniques to reduce nesting:**
- **Guard clauses** — handle the edge case and return immediately
- **No `else` after `return`** — if the `if` block returns, the rest of the function
  *is* the else
- **Extract function** — pull the nested block into a named function
- **Invert the condition** — flip `if ok { ...long block... }` to
  `if !ok { return }`

```go
// Bad
func (h *Handler) status(order *Order) string {
	if order != nil {
		if order.Shipped {
			return "shipped"
		} else {
			return "processing"
		}
	} else {
		return "not_found"
	}
}

// Good
func (h *Handler) status(order *Order) string {
	if order == nil {
		return "not_found"
	}
	if order.Shipped {
		return "shipped"
	}
	return "processing"
}
```

### Extract Complex Conditions

When an `if` has a compound condition, extract it into a well-named function or
variable. The name should describe the *business meaning* of the condition, not its
mechanics.

```go
// Bad — the reader has to parse the logic to understand the intent
if order.PlacedAt.Add(refundWindow).Before(time.Now()) || order.Status == StatusRefunded {
	return errNotRefundable
}

// Good — the intent is immediately clear
if isPastRefundWindow(order) || isAlreadyRefunded(order) {
	return errNotRefundable
}
```

This applies regardless of condition length. Even a two-part condition is worth
extracting if the business meaning isn't obvious from the raw expressions.

### No Hidden Side Effects

A function named `checkToken` should not also record an audit entry. If it has side
effects, the name must communicate them — or better, split it into two functions.

### Context

- `ctx context.Context` is the first parameter of anything that does I/O, blocks, or
  calls something that does.
- Never store a `Context` in a struct. Pass it down the call chain.
- Always propagate the caller's context. `context.Background()` belongs in `main` and
  tests, not in the middle of a request.

## No Unnecessary Ceremony

Write the simplest form that communicates the intent. Extra syntax that adds nothing
is noise.

- **`:=` over `var x T = ...`.** Use `var` when you want the zero value (`var buf bytes.Buffer`).
- **Make the zero value useful.** A struct that works without a constructor doesn't
  need one.
- **Keyed struct literals** for any struct type you didn't define in the same package.
- **No naked returns.** Named results are fine for documentation; returning them
  implicitly is not.
- **No getters and setters for their own sake.** Export the field or don't.
- **A nil slice is a perfectly good empty slice.** Don't allocate `[]T{}` to avoid nil.
- **No `fmt.Sprintf` for plain concatenation.** `"Bearer " + token` is fine.

### No Ceremony for Tooling's Sake

If only a *linter* demands a piece of syntax, configure the linter. If the *compiler*
demands it, it stays. We don't write code whose only reader is a tool we control.

```go
// Noise — Go does not require this. It exists only to silence a linter.
_ = response.Body.Close()
_, _ = w.Write(payload)

// Clean — a call statement may discard its results
response.Body.Close()
w.Write(payload)
```

Configured away rather than written:
- `_ = f()` / `_, _ = f()` for calls whose errors are never actionable
  (`Close` on a response body, `ResponseWriter.Write`, `fmt.Fprint*`). These go on the
  linter's exclusion list. The error check stays **on** everywhere else — silently
  dropping a meaningful error is a bug, not a style choice.
- Renaming unused parameters to `_`.
- Mandatory doc comments on packages and exported identifiers.
- Scattered `//nolint` directives. If a rule is wrong for this project, turn it off in
  the config, once.
- `var _ Iface = (*T)(nil)` assertions when the type is already used as that interface
  somewhere the compiler checks.

Forced by the compiler — accept it and move on:
- `x, _ := f()` when you need `x` and not the second value. (If that second value is
  an `error`, see Error Handling — ignoring it needs a reason.)
- Unused imports and unused local variables are errors.
- Explicit numeric conversions, trailing commas, braces on every block.

## Magic Values

Inline numbers and strings with no context are unreadable. Give them a name.

```go
// Bad — what is 24*30? What is 100?
if order.PlacedAt.Add(time.Hour * 24 * 30).Before(time.Now()) { ... }
if len(order.Items) > 100 { ... }

// Good
const refundWindow = 30 * 24 * time.Hour
const maxItemsPerOrder = 100
```

Strings that come from an API's or schema's enum are magic values too. `"pending"`,
`"shipped"`, `"cancelled"` get typed constants, defined once.

Exceptions: `0`, `1`, `-1`, `""`, and `nil` are usually self-evident in context and
don't need names. `http.StatusOK` and friends already have names — use them. Use
judgment — if the meaning is obvious, don't over-name it.

## Comments

### When Comments Are Warranted

- **Why, not what.** Explain business reasons, tradeoffs, or non-obvious constraints.
- **Legal, regulatory, or spec requirements** — including which requirement a rule
  implements (`// RFC 9110 §15.5.10: ...`, `// PCI DSS 3.4: ...`).
- **Warnings** — `// WARNING: called from multiple goroutines`
- **TODOs** — see below.

Doc comments on packages and exported identifiers are **not required**. The compiler
doesn't ask for them, and a good name beats a sentence restating it. When a comment *is*
warranted on an exported identifier, write it in godoc form — starting with the name,
directly above the declaration — so `go doc` and editors surface it:

```go
// ErrDeclined reports that the payment provider refused the charge.
// Callers map it to a 402 response, never to a 500.
var ErrDeclined = errors.New("payment: declined")
```

### TODOs

Write them as a bare `// TODO: ...` that says what's missing and why it matters. Don't
put issue IDs in comments — the tracker is where the work lives; the comment is where
the reader needs the context.

### When Comments Are Code Smells

- Explaining what the code does → rename things instead
- Commented-out code → delete it. Git remembers.
- Journal comments or change logs → that's what git log is for
- Section banners (`// ---- helpers ----`) → the file is too long or too mixed; split it

## Dead Code

Dead code is a liability. It bloats the source, bloats the tests, widens the attack
surface, and misleads anyone reading it into thinking it matters. If a function, type,
branch, or constant has no call sites and no reason to exist:

1. Delete the dead code
2. Delete any tests that only covered the dead code
3. Verify the remaining test suite still passes

The compiler already rejects unused imports and locals. It does *not* catch unused
functions, methods, types, or exported identifiers nobody imports — those are on you.

Commented-out code is dead code. Unreachable branches (a `case` no value ever hits, an
`else` that can never trigger) are dead code. Delete them. Do not keep dead code "just
in case." Version control exists for that.

## Error Handling

Errors are values. There are no exceptions to catch.

- **Return `error` as the last result.** Check it immediately; keep the happy path
  unindented.
- **Add context when you pass an error up**, and wrap with `%w` so callers can still
  match it: `fmt.Errorf("save order %s: %w", id, err)`.
- **Match errors by identity or type, never by string.** `errors.Is(err, ErrNotFound)`,
  `errors.AsType[*ConflictError](err)`.
- **Handle an error once.** Either log it or return it. Doing both produces the same
  failure three times in the logs.
- **Error strings are lowercase with no trailing punctuation**, prefixed with the
  package when it helps: `"auth: scope is empty"`.
- **`panic` is for programmer errors only** — a broken invariant, an impossible state.
  Bad input is not a programmer error. `Must*` functions belong in package-level
  initialization and tests, not on a request path.
- **Don't ignore an error without a reason.** `x, _ := f()` on an error needs either a
  one-line comment explaining why it cannot fail here, or a `TODO`.
- **"Not found" is not nil.** Prefer `(T, bool)` or `(T, error)` with a sentinel
  (`ErrNotFound`) over returning a nil pointer the caller must remember to check.
- **Don't pass nil** as an argument when the function doesn't expect it.
- **Error handling is one thing.** A function that translates errors into responses
  should do little else. Extract the happy-path logic from the error mapping.

## Concurrency

- **Every goroutine has an owner and a way to stop.** If you can't say who waits for it
  and what cancels it, don't start it.
- **Shared state is guarded.** A map touched by concurrent HTTP handlers needs a mutex
  or must be confined to one goroutine. Document which with a comment on the field.
- **The sender closes the channel.** Never the receiver.
- **`go test -race` is mandatory.** A race the detector reports is a bug, even if the
  test passed.

## Structure and Organization

### The Package Is the Unit

Go has no "one export per file." A *package* has one purpose; its files are just
chapters.

- **One package, one purpose.** If you need "and" to describe the package, split it.
- **Name files for their primary type or operation** — `create.go`, `cancel.go`,
  `order.go`. A reader should be able to guess the file from the behavior.
- **No grab-bag packages.** `common`, `helpers`, `misc`, `shared` say nothing about
  what's inside and become dumping grounds. Name the package for what it provides, or
  put the function next to its only caller.
- **Generic helpers get stdlib-style extension packages**, named for the subject they
  extend — `slicesx`, `stringsx`, `jsonx`. They hold only small, domain-free helpers
  that remove standard-library ceremony (`Map`, `IsBlank`). Nothing in them may know
  what the application's domain is; the moment a helper does, it moves next to its
  caller.
- **Package names are short, lowercase, singular-ish nouns** with no underscores:
  `orders`, `billing`, `payment`.

### The Newspaper Metaphor

Read a file top to bottom like a newspaper article: types first, then constructors,
then exported functions and methods, then the unexported helpers they call. Callers
above callees.

### Vertical Distance

Things that are related should be close together. If function A calls function B, they
should be near each other in the file. Don't make the reader jump around.

### Line Length

Keep lines under ~100 characters. `gofmt` never wraps lines, so this one is on you. A
long line is usually a sign the expression is doing too much — extract a variable, put
each argument on its own line, or pull logic into a named function.

Exceptions where a longer line is acceptable:
 - **Import paths**
 - **URLs in comments** — you can't break a URL across lines
 - **String literals** — error and log messages where breaking mid-sentence hurts more
   than the long line does
 - **Struct tags**
 - **Function signatures** that would be less readable broken than whole — though four
   long parameters is its own smell

The common thread: content that is inherently one unit, where breaking it creates more
noise than the long line does.

### File Length

Aim for files under ~200 lines. Files over 400 almost certainly hold more than one
chapter and should be split. Test files and generated files are exempt from the
number, not from the principle.

### Screaming Architecture

The package tree should scream what the application *does*, not what framework it uses.
A glance at the directories should tell you the domain — orders, billing, inventory,
customers — not the technical layers.

```
// Bad — screams "I'm a web service"
controllers/
models/
services/
utils/

// Good — screams "I'm a store"
orders/
billing/
inventory/
customer/
shipping/
```

Group by domain concept. The handler, its request types, its validation, and its tests
for orders live in `orders/` — not scattered across four layer packages.

### Organized Imports

`gofmt` sorts imports within a group. We use three groups, separated by blank lines:

1. **Standard library**
2. **Third-party modules**
3. **This module** — the path declared in `go.mod`

```go
import (
	"context"
	"net/http"
	"time"

	"github.com/google/uuid"

	"example.com/shop/internal/billing"
	"example.com/shop/internal/orders"
)
```

Sorted, grouped imports are scannable, diffable, and make duplicates obvious. An agent
adding an import to a sorted group will place it correctly.

### The Boy Scout Rule

Leave every file cleaner than you found it. If you touch a file to make a change,
improve one small thing: a name, a comment, a simplification. Over time, the codebase
gets better instead of worse.

## DRY — But Not Prematurely

Duplication is acceptable when:
- You've only seen the pattern once (it might not actually be a pattern)
- The two instances might diverge as requirements evolve
- The abstraction would be harder to understand than the duplication

Extract a shared abstraction after you see the same pattern THREE times. At that point,
you understand what varies and what doesn't. Go's proverb says the same thing: *a
little copying is better than a little dependency.*

## SOLID at a Glance

- **Single Responsibility** — a package has one reason to change.
- **Open/Closed** — extend behavior by adding an implementation of an interface, not by
  editing a `switch` in five places.
- **Liskov Substitution** — any implementation of an interface must honor the whole
  contract, including its error behavior. A fake that can't fail where the real thing
  can is lying to its tests.
- **Interface Segregation** — Go's one- and two-method interfaces are this principle
  made idiom. **Define the interface where it is consumed**, with only the methods that
  consumer calls. Don't export a twelve-method interface next to its only
  implementation.
- **Dependency Inversion** — accept interfaces, return structs. Inject dependencies as
  struct fields (`Handler{Payments: ..., Store: ...}`) so production and tests wire the
  same type differently.

Don't introduce an interface until there is a second implementation. An in-memory
implementation used by tests counts as a second implementation.

You don't need to cite these principles by name. Just follow them.

## Agent-Friendly Code

These rules are good practice for humans too, but coding agents benefit from them
disproportionately. Agents pattern-match aggressively, read code literally, and
struggle with indirection. The cleaner and more consistent the code, the better agents
work with it.

### Consistency Over Preference

If the codebase does something one way, do it that way everywhere — even if you prefer
another style. Three different patterns for the same thing (three ways to write a JSON
response, three error shapes, three ways to build a test handler) means an agent will
pick one arbitrarily or mix them. One pattern, used everywhere, means the agent learns
it once and applies it correctly.

When in doubt, grep for precedent before introducing a new pattern.

### Colocate Context

If understanding a function requires reading 5 other packages, an agent has to load all
of them — and might not. Self-contained functions with minimal distant dependencies are
easier for agents to work with correctly. This reinforces screaming architecture:
keeping a feature's handler, validation, and tests in the same package means the agent
doesn't have to hunt.

### No Clever Code

Agents take code literally. Obscure language tricks, overly dense one-liners, or
"smart" generic gymnastics that require deep expertise to parse will trip them up more
than they'd trip a human. If a junior developer would need to stop and think about it,
an agent will probably misread it.

Write obvious code. Save your cleverness for the architecture.

### Avoid Untraceable Dispatch

Interfaces are Go's designed mechanism for dispatch, and they are fine — `gopls` can
find every implementation. What agents (and humans) *can't* statically trace:

- `reflect`-driven behavior
- `any` and `map[string]any` where a struct would do — the fields become invisible to
  the compiler and to search
- `init()` functions with side effects, and registration-by-import
- Package-level mutable state
- String-keyed lookups that stand in for method calls
- `//go:linkname` and `unsafe`

If you must use one, document it explicitly — but prefer static, typed, traceable
calls.
