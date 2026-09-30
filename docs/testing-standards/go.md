# Testing Standards

Reference guide for test quality. Read this before writing any tests. For the Go
mechanics — file layout, assertions, test doubles, commands — see
`/docs/testing-setup.md`.

## Test Naming

Test names are specifications. They describe behavior, not implementation. They should
read like documentation.

Go test functions are identifiers, so the sentence is written in `MixedCaps`. Subtests
take plain strings.

**Good:** `TestRejectsExpiredTokens`, `TestReturnsEmptyListWhenNoOrdersMatch`,
`t.Run("rejects a refund past the refund window", ...)`
**Bad:** `TestValidate`, `TestPut2`, `TestItWorks`, `TestHandler_Success`

Aim for the pattern `<expected behavior> when <condition>` — but prioritize readability
over rigid format. Don't prefix every test with the name of the function under test;
the package and file already say where you are.

## Test Structure

Every test follows Arrange → Act → Assert:

```go
// Arrange — set up the preconditions
// Act — execute the behavior under test
// Assert — verify the outcome
```

Go has no `before`/`after` hooks, and that's fine: extract shared setup into small
builder functions (`newHandler()`, `newOrderBody()`) and call them explicitly.
When a builder handles the Arrange, the test body can focus on the Act and the
assertions — this is a good thing. Register teardown with `t.Cleanup` inside the
builder so the test never has to remember it.

Keep tests readable. If you can't tell what a test does without reading three different
builders, the setup has been over-extracted.

## One Behavior Per Test

Each test verifies ONE logical behavior. Multiple assertions are fine if they all
verify the same behavior from different angles. But if a test fails, you should
immediately know WHAT broke without reading the test body.

**Good:** Several asserts checking that a created order has the right status, total,
AND items (one behavior: order creation)
**Bad:** One test that creates an order, then pays for it, then cancels it (three
behaviors)

### Table-Driven Tests

A table is **one behavior over many inputs** — not many behaviors in one function.

- Every case has a name that reads as a specification. It becomes the subtest name.
- No branching inside the loop. If the body needs `if tc.wantErr { ... } else { ... }`,
  you have two behaviors: write two tables, or two tests.
- If a table has one row, it's not a table. Write a plain test.

## Test Behavior, Not Implementation

Tests should describe WHAT the system does, not HOW it does it internally. Test from
the outside in: call exported functions, send requests to handlers, call the client
against a test server. Check the outputs and side effects — not the internal steps that
produced them.

This is why tests live in an external test package (`package orders_test`) by
default: the compiler then guarantees the test can only touch what a real caller could.

**Signs you're testing implementation:**
- The test needs an unexported function, field, or type
- Asserting on internal state that isn't part of the public contract
- Tests that break when you refactor without changing behavior
- Asserting the exact sequence of calls made to a collaborator
- A test double that records calls, where checking the resulting state would do

**Signs you're testing behavior:**
- Tests use only the exported API or the HTTP surface
- Refactoring internals doesn't break tests
- Test names read as specifications a user or API consumer would recognize
- Tests would still make sense if you rewrote the implementation from scratch

### Layers of "Outside-In"

Outside-in is not all-or-nothing. There are layers:

- The **HTTP wiring** (routes, middleware) is separate from the **logic** each route
  invokes. These can and should be tested independently.
- **Shared code** that is implicitly tested through its callers may deserve direct
  tests if it has grown into its own package with its own responsibilities. Promoting
  it to a directly testable unit means its dependents can substitute it, keeping their
  tests simpler.
- The question is: does this code have its own contract? If yes, test it directly. If
  it's an unexported helper that only exists to serve one caller, implicit coverage is
  fine.

## Test Independence

- No test may depend on another test's execution or state
- No test may depend on execution order
- Every test builds its own state; `t.Cleanup` tears it down
- Shared fixtures are fine, but shared MUTABLE state is not. No package-level variables
  that tests write to.

## No Duplicate Tests

Before writing a new test, search the test suite for existing tests that cover the same
behavior. Duplication creates noise, false confidence, and maintenance burden.

- If a test already exists for the behavior, **update it** if the behavior has changed
- If two tests cover the same behavior, **remove the weaker one** (less descriptive
  name, fewer edge cases, more coupled to implementation)
- If a behavior has been removed, **remove the test(s)** for that behavior

Duplicate tests often appear after refactoring when old tests are left behind. Clean
these up proactively.

## Implicit Coverage Is Real Coverage

Code does not need its own dedicated test file to be considered tested. A helper called
by a handler is tested through that handler's tests. A validation function used by one
operation is tested through that operation's tests.

Before declaring code "untested":
1. Trace the call sites
2. Check whether existing tests exercise the code path
3. Only write a new test if the behavior is genuinely uncovered

**Exception:** Shared code that has grown into its own package — with its own
responsibilities and multiple consumers — should be promoted to a directly testable
unit. This lets you test its contract in isolation and substitute it in dependent
tests, keeping those tests simpler and more focused.

Write dedicated tests for shared packages or complex logic that benefits from isolated
edge-case testing. But do not create a `_test.go` for every `.go` as a goal unto itself.

## Dead Code

Dead code is a liability. It bloats the source, bloats the tests, and widens the attack
surface. If a function, type, or branch has no call sites and no reason to exist:

1. Delete the dead code
2. Delete any tests that only covered the dead code
3. Verify the remaining test suite still passes

Do not keep dead code "just in case." Version control exists for that.

## Shared Test Data

Most applications have a core domain that shows up in nearly every test — customers,
orders, and products in a store; accounts and transactions in a ledger. Rather than rebuilding this data
from scratch in every test file, define a shared set of well-known test entities that
the entire suite can reference.

### Named Fixtures

Give test entities recognizable names and fixed roles. When someone reading a test sees
the same repeat customer or the same out-of-stock product they saw in the last file,
they immediately know what it represents — it's not some throwaway literal.

### Guidelines

- Define fixtures as **functions that return a fresh value** (`NewOrder()`), never
  as package-level variables. A function can't leak mutations between tests.
- Give each fixture a clear, memorable identity: what it is, and whatever attributes
  matter to the domain — status, amount, who owns it. Name it for the
  role it plays, not its contents.
- Keep the set small and stable — a handful of well-known fixtures is better than
  dozens of forgettable ones
- Tests that need a truly unusual entity (edge cases, specific error conditions) should
  build their own, or take a fixture and change the one field that matters. The
  difference from the well-known fixture *is* the test's subject — make it visible.
- Time-dependent fixtures are relative to "now" (an order placed an hour ago), never
  hard-coded dates that rot.

### Why This Matters

Uniform test data makes the suite read like a cohesive story rather than a collection
of isolated fragments. A new developer scanning test files picks up the cast quickly.
That familiarity reduces cognitive load and makes tests easier to write, read, and
review.

## What to Test

- **Happy path** — the expected, normal usage
- **Edge cases** — empty inputs, nil, empty collections, boundary times, off-by-one
- **Error cases** — invalid input, a collaborator that fails, a dependency that times
  out
- **Integration between packages** — the handler may work against an in-memory store,
  but does the real client serialize what the remote service expects?
- **Security boundaries** — see `/docs/security-checklist.md` and
  `/docs/validation-boundaries.md`

## What NOT to Test

- Things that already have tests (see "No Duplicate Tests")
- The standard library or third-party library behavior
- Generated code (files marked `// Code generated ... DO NOT EDIT.`)
- Implementation details that aren't part of the public contract
- Thin wrappers that only delegate to a library — these are covered by integration
- Dead code (delete it instead)

## Testing HTTP Services

There are two layers, and they are tested separately:

**Logic tests** verify behavior — that given certain input, the operation produces the
correct output and side effects. Call the handler method directly with an
`httptest.ResponseRecorder`, or better, call the transport-free function underneath it.
No router, no server, no middleware.

**Route tests** verify the HTTP wiring — that the right methods and paths reach the
right handlers, and that middleware (auth, logging) is applied. Route tests assert only
on *which handler is registered* for a method and path — without running it. They should
**not** assert on response bodies or status codes that belong to the handler's contract.

This separation means changes to handler logic only break logic tests, and changes to
routing only break route tests. If route tests assert on real responses, a handler
change breaks both layers — exactly the kind of coupling we want to avoid.

## Assertion Style

- Use concrete expected values when the specific value is the behavior under test. If
  the store should hold exactly 1 order, assert `1` — that's a behavioral claim.
- For random or generated values (UUIDs, tokens), assert on shape or validity rather than
  exact values.
- For times, assert within a tolerance or control the clock — never assert equality
  against the wall clock.
- For collections, assert on specific contents when the values matter, and on length or
  emptiness when they don't.
- For errors, assert on identity or type (`ErrorIs`, `ErrorAs`) — never on the message
  text, unless the message *is* the contract.
- Use the most specific assertion available. It produces the failure message you'll
  want at 2am.

The guiding question: **is the exact value part of the behavior contract, or am I just
checking that something reasonable came back?** Let that answer drive the assertion.

## Coverage Philosophy

We value meaningful coverage of critical paths over chasing a percentage. A codebase
with 60% coverage of the right things is better than 95% coverage padded with trivial
tests.

Focus coverage on:
1. Business logic and domain rules — the rules the service exists to enforce
2. Security and validation boundaries
3. Error handling and failure modes
4. Complex conditional logic

Do not write tests solely to increase a coverage number. Every test must protect
against a real regression.

**Mutation testing is how we check that.** Line coverage says a line ran; a surviving
mutant says no test would notice if that line were wrong. A mutant that survives in
business logic is a missing test. A test that kills no mutants is a candidate for
deletion.

## Test Speed

- Unit tests must be fast. If a test hits a real network, disk, or database, it's an
  integration test — isolate it.
- Prefer in-memory implementations over mocks. Mocks verify interaction; fakes verify
  behavior.
- If you must substitute, substitute at the boundary (HTTP, clock, storage) — never
  the code under test.
- Never `time.Sleep` to wait for something. Control the clock or wait on a signal.

## Red → Green → Refactor Checklist

Before moving from GREEN to REFACTOR, ask:
- [ ] Is there duplication between the new test and existing tests? Extract shared setup or remove the duplicate.
- [ ] Can the test name be more descriptive?
- [ ] Is the test coupled to implementation details?

Before moving from REFACTOR to the next RED, ask:
- [ ] Are all tests still green, with `-race`?
- [ ] Is the production code as simple as it can be for the behaviors tested so far?
- [ ] Did I note this cycle as a commit candidate for handoff? (Agents don't commit.)
