# Testing Setup — Clojure + ClojureScript

Stack-specific testing conventions for Clojure (backend) and ClojureScript (frontend) projects. For general testing philosophy, see `/docs/testing-standards.md`.

## Framework

- **Clojure backend:** `speclj.core`
- **ClojureScript frontend:** `speclj.core` with `c3kit.scaffold.cljs` as the test runner

Never use `clojure.test` or `cljs.test` — all tests use Speclj.

## File Organization

Source and spec files are separated by platform. Shared code uses `.cljc` and lives in `src/cljc` (source) and `spec/cljc` (specs). Spec files use a `_spec` suffix.

```
src/
  clj/
    myapp/
      handler.clj
  cljc/
    myapp/
      common.cljc
  cljs/
    myapp/
      home.cljs
spec/
  clj/
    myapp/
      handler_spec.clj
      spec_helper.clj            ;; Clojure-only test helpers
  cljc/
    myapp/
      common_spec.cljc
      spec_helperc.cljc          ;; cross-platform test helpers
  cljs/
    myapp/
      home_spec.cljs
      spec_helper.cljs           ;; ClojureScript-only test helpers
```

Not every source file needs a corresponding spec file. If `common.cljc` is fully exercised through the specs of its consumers, a separate `common_spec.cljc` is unnecessary — unless it has grown into its own module with its own responsibilities (see "Implicit Coverage" in `/docs/testing-standards.md`).

## Test Structure

Each spec file has one top-level `describe` block. Use `it` for individual test cases and `context` for logical groupings within the describe.

```clojure
(ns myapp.handler-spec
  (:require [speclj.core :refer :all]
            [myapp.handler :as sut]))

(describe "Handler"

  (context "creating a post"

    (it "rejects requests missing a title"
      (let [result (sut/create-post {:body "Hello"})]
        (should= :invalid (:status result))
        (should= "Title is required" (-> result :errors :title))))

    (it "builds the entity from user input and server-controlled fields"
      (let [result (sut/create-post {:title "My Post" :body "Hello"})]
        (should= :created (:status result))
        (should-not-be nil? (:id (:entity result)))
        (should= "draft" (:status (:entity result))))))

  (context "with an expired token"

    (it "rejects the request"
      (let [token (make-token {:exp (- (now) 3600)})
            result (sut/validate-token token)]
        (should= :invalid (:status result))))))
```

### Setup and Teardown

Use `before`, `after`, `before-all`, and `after-all` to manage shared setup and teardown. These keep test bodies clean and eliminate duplication — often removing the need for an explicit Arrange or Act section in each test.

```clojure
(describe "User service"

  (before-all
    (init-test-db))

  (before
    (clear-users))

  (after-all
    (teardown-test-db))

  (it "creates a user with the given name"
    (let [user (sut/create-user {:name "Alice"})]
      (should= "Alice" (:name user)))))
```

## Assertions

| Pattern | Speclj |
|---------|--------|
| Equality | `(should= expected actual)` |
| Inequality | `(should-not= unexpected actual)` |
| Predicate (truthy) | `(should-be pos? x)` |
| Predicate (falsy) | `(should-not-be nil? x)` |
| Exception | `(should-throw ExceptionType (expr))` |
| Invocation | `(should-have-invoked :fn-var)` |

`should-be` and `should-not-be` always take a predicate function — they do not accept bare values.

Speclj does not support inline message strings on assertions like `clojure.test` does. Use descriptive `it` and `context` strings to make failure output clear — the test name is the message.

## Test Helpers

Test helpers are separated by platform:

- `spec/clj/myapp/spec_helper.clj` — Clojure-only helpers (backend test data, DB setup/teardown)
- `spec/cljs/myapp/spec_helper.cljs` — ClojureScript-only helpers (DOM utilities, rendering helpers)
- `spec/cljc/myapp/spec_helperc.cljc` — cross-platform helpers (shared factories, common assertions)

Common helpers include factory functions for test data (`make-user`, `make-token`), request builders for API tests, and database setup/teardown utilities.

## Backend API Testing

Handler functions can be tested directly — no need to go through the full HTTP stack for every test case. This keeps handler tests focused on business logic.

**Route tests** are separate. They verify that the router wires the correct HTTP methods and paths to the correct handlers, and that middleware (auth, validation, error handling) is applied. Route tests should mock the handler function and use `should-have-invoked` to verify the wiring — they should not assert on response bodies or status codes that belong to the handler's contract.

```clojure
;; Handler test — tests business logic directly
(describe "create-post handler"
  (it "returns the created post with a generated ID"
    (let [result (sut/create-post {:title "Hello" :body "World"})]
      (should= :created (:status result))
      (should-not-be nil? (:id (:entity result))))))

;; Route test — tests wiring only, mocks the handler
(describe "POST /api/posts"
  (it "routes to the create-post handler"
    ;; mock create-post handler, make request, verify invocation
    (should-have-invoked :myapp.handler/create-post)))
```

This separation means changes to handler logic only break handler tests, and changes to routing only break route tests.

## ClojureScript Component Testing

Test Reagent components by rendering them and asserting on the output. Test through events and DOM state — not by calling component functions directly or inspecting atoms.

## Mocking

- Prefer `with-redefs` for overriding functions at I/O boundaries (database, HTTP, clock)
- Never `with-redefs` the function under test
- For database-dependent tests, prefer an in-memory database or test fixtures with transaction rollback
- For time-dependent tests, override the clock function — don't depend on wall-clock time
- For shared code promoted to a testable module, fake it in dependent tests rather than using the real implementation
- For route tests, mock the handler and use `should-have-invoked` — route tests verify wiring, not logic

## Running Tests

```bash
# Clojure — run once
clj -M:test:spec

# Clojure — auto-run on file change
clj -M:test:spec -a

# ClojureScript — run once
clj -M:test:cljs once

# ClojureScript — auto-run on file change
clj -M:test:cljs

# Babashka — run once
bb spec

# Babashka — auto-run on file change
bb spec -a
```

Ensure these commands are documented in the project README and that at least the run-once variants are used in CI.