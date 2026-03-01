# Clean Code Standards — Clojure + ClojureScript

Clean code is a UX problem. Our users are other developers, coding agents, and our
future selves — we're creating a user experience for people who read our code.

Like art, code quality is almost always subjective, but you can tell good code from bad
code because the good code follows rules. This document defines those rules. Any of
them can be broken — but only with reason and intention. Breaking a rule because you
thought about it and decided the code is clearer without it is clean code. Breaking a
rule because you didn't know it existed is not.

## Naming

Names are the most important tool for readability. Invest time in them.

- **Vars and bindings** — describe what they hold, not their type. `remaining-attempts` not `n` or `x`.
- **Functions** — describe what they do with a verb. `calculate-discount` not `discount` or `do-stuff`.
- **Predicates** — end with `?`. `expired?`, `has-permission?`, `should-retry?`.
- **Constants / def values** — describe the meaning, not the value. `max-login-attempts` not `three`.
- **Avoid abbreviations** unless universally understood (`url`, `id`, `http` are fine; `usr`, `mgr`, `ctx` are not).
- **Avoid generic names** — `data`, `info`, `item`, `result`, `temp`, `val` almost always have a better name.

If you need a comment to explain what a var or function does, the name is wrong.
Rename it.

### Language Conventions Win

When a Clojure convention conflicts with a general clean-code heuristic from another
language, the Clojure convention wins. `kebab-case` for everything. `?` suffix for
predicates. `!` suffix for side-effecting functions. `*earmuffs*` for dynamic vars.
These conventions aren't just style — they communicate semantics and are expected by
every Clojure developer and tool.

## Functions

### Do One Thing

A function does one thing if you cannot meaningfully extract another function from it.
If you can describe it only with "and" or "then," split it.

### Keep Them Short

Aim for ~10 lines. This isn't a hard ceiling — some functions will be longer and that's
fine — but if a function exceeds 10 lines, look for extraction opportunities. If it
exceeds 20, it almost certainly does more than one thing.

Extract helpers with `defn-` (private) for implementation details that don't need to
be part of the namespace's public API.

### Limit Parameters

- 0-2 parameters: ideal
- 3 parameters: acceptable, consider a map
- 4+ parameters: refactor into a map or split the function

Clojure's convention of passing option maps is a natural fit here:

```clojure
;; Bad — what do these arguments mean at the call site?
(create-user "Alice" "alice@example.com" :admin true 30)

;; Good — a map is self-documenting
(create-user {:name  "Alice"
              :email "alice@example.com"
              :role  :admin
              :age   30})
```

### Phantom Parameters

If a parameter always receives the same value at every call site, it's not a real
parameter — it's a phantom. Hard-code it into the function and remove it from the
signature. The parameter can always be reintroduced later if requirements actually
demand it. Until then, it's noise that every caller has to know about for no reason.

```clojure
;; Every caller passes :utf-8 — it's not a real choice
(read-file path :utf-8)
(read-file other-path :utf-8)

;; Just hard-code it
(defn read-file [path]
  (slurp path :encoding "UTF-8"))
```

### No Flag Arguments

A boolean parameter that switches a function between two behaviors means the function
does two things. Split it.

```clojure
;; Bad — what does `true` mean here?
(process-order order true false)

;; Good — two functions with clear names
(ship-order order)
(hold-order order)
```

If you see `(do-thing x true)` or `(do-thing x false)` and can't tell what the boolean
controls without reading the implementation, that's a flag argument. Extract it into
separate functions with descriptive names.

### Minimize Nesting

Keep nesting shallow. Deeply nested `let`, `if`, `when`, and `cond` forms obscure
logic.

**Techniques to reduce nesting:**
- **Early return with `when` / `when-not`** — handle the edge case and short-circuit
- **Extract method** — pull the nested block into a named function
- **Threading macros** — flatten nested function calls (see below)

### Extract Complex Conditions

When a `cond` or `if` has a compound condition, extract it into a well-named predicate
function. The name should describe the *business meaning*, not the mechanics.

```clojure
;; Bad — the reader has to parse the logic
(when (or (= (:role user) :admin)
          (and (= (:role user) :editor)
               (= (:author-id post) (:id user))))
  ;; ...
  )

;; Good — the intent is immediately clear
(defn- can-edit-post? [user post]
  (or (admin? user)
      (author? user post)))

(when (can-edit-post? user post)
  ;; ...
  )
```

This applies regardless of condition length. Even a two-part condition is worth
extracting if the business meaning isn't obvious from the raw expressions.

**Avoid deeply nested `cond` / `if` trees.** If the branching logic is complex, extract
it into a named function. A `cond` with 6+ branches may indicate a need for a
multimethod or a lookup map.

### No Side Effects

A function named `check-password` should not also initialize a session. If it has side
effects, the name must communicate them: `check-password-and-init-session!` — or
better, split it into two functions. Use the `!` suffix to signal side effects.

### Use Threading Macros

Use `->` and `->>` to flatten nested function calls. Nested calls read inside-out;
threading reads top to bottom.

```clojure
;; Nested — hard to read
(filter active? (map normalize (remove nil? (get-users db))))

;; Threaded — reads top to bottom
(->> (get-users db)
     (remove nil?)
     (map normalize)
     (filter active?))
```

If a pipeline exceeds 5-6 steps, consider whether intermediate values with descriptive
`let` bindings would be clearer.

### Use Descriptive `let` Bindings

Use `let` to name intermediate results, especially when a single expression would be
dense. The binding name is documentation.

Align `let` binding values vertically, the same way you would align map values. This
makes it easy to scan the names and values independently.

```clojure
;; Dense — reader has to unpack it
(send-email (format-welcome (:email (find-user db user-id))))

;; Named and aligned — each step is self-documenting
(let [user          (find-user db user-id)
      email-address (:email user)
      message       (format-welcome email-address)]
  (send-email message))

;; Bad — unaligned values are harder to scan
(let [user (find-user db user-id)
      email-address (:email user)
      message (format-welcome email-address)]
  (send-email message))
```

This alignment rule applies to all binding forms: `let`, `if-let`, `when-let`, `loop`,
and similar.

## Idiomatic Clojure

**`when` over `(if x y nil)`.** Use `when` when there's no else branch. It
communicates intent (side-effect or conditional inclusion) and avoids a dangling `nil`.

**Keyword access over `get` for simple lookups.** `(:name user)` reads more naturally
than `(get user :name)`. Use `get` when you need a default value:
`(get user :name "Unknown")`.

**`some?` / `nil?` over equality checks.** `(some? x)` is clearer than
`(not (nil? x))`. Use the predicate that names your intent.

## No Unnecessary Ceremony

Write the simplest form that communicates the intent. Extra syntax that adds nothing
is noise.

**Map formatting.** Expand vertically when there are 3+ key-value pairs. One or two
short pairs can stay on one line. Use visual judgment — two pairs with long values
should expand. Be consistent within a single map: either everything on one line or
everything expanded. Never mix.

```clojure
;; Fine — two short pairs
{:x 10 :y 20}

;; Expand — three or more
{:name  "Alice"
 :email "alice@example.com"
 :role  :admin}

;; Bad — inconsistent. Pick one.
{:name "Alice" :email "alice@example.com"
 :role :admin}
```

**Align map values.** When a map expands vertically, align the values for scannability.
This makes it easy to read down the column of keys or values independently.

```clojure
;; Scannable
{:name   "Alice"
 :email  "alice@example.com"
 :role   :admin
 :active true}

;; Harder to scan
{:name "Alice"
 :email "alice@example.com"
 :role :admin
 :active true}
```

This isn't about being clever or terse. It's about removing visual noise so the
reader's attention goes to what the code *does*, not how it's punctuated.

## Magic Values

Inline numbers and strings with no context are unreadable. Give them a name.

```clojure
;; Bad — what is 86400000? What is 3?
(schedule-reconnect 86400000)
(when (> attempts 3) (lock-account account))

;; Good
(def one-day-ms 86400000)
(def max-login-attempts 3)

(schedule-reconnect one-day-ms)
(when (> attempts max-login-attempts) (lock-account account))
```

Exceptions: `0`, `1`, `-1`, empty string, and `nil` are usually self-evident in context
and don't need names. Use judgment — if the meaning is obvious, don't over-name it.

## Comments

### When Comments Are Warranted

- **Why, not what.** Explain business reasons, tradeoffs, or non-obvious constraints.
- **Legal or regulatory requirements.**
- **TODOs with a bead ID** — `; TODO(bd-42): Handle rate limiting` ties the comment to tracked work.
- **Warnings** — `; WARNING: This is called from multiple threads`

### When Comments Are Code Smells

- Explaining what the code does → rename things instead
- Commented-out code → delete it. Git remembers.
- Journal comments or change logs → that's what git log is for

## Dead Code

Dead code is a liability. It bloats the source, bloats the tests, and misleads anyone
reading it into thinking it matters. If a function, namespace, branch, or var has no
call sites and no reason to exist:

1. Delete the dead code
2. Delete any tests that only covered the dead code
3. Verify the remaining test suite still passes

Commented-out code is dead code. Delete it. Git remembers.

Unreachable branches (a `cond` clause that can never match, an `if` branch that never
triggers) are dead code. Delete them.

Do not keep dead code "just in case." Version control exists for that.

## Error Handling

- **Null is fine.** Returning `nil` is often the cleanest signal that nothing was found. Don't wrap it in an empty collection unless the caller actually benefits from it.
- **Don't pass nil** as a function argument when the function doesn't expect it.
- **Error handling is one thing** — a function that handles errors should do little else. Extract the "happy path" logic from the error handling logic.
- **Use `ex-info` for rich exceptions** — attach context as data rather than encoding it into message strings.

```clojure
;; Good — structured error data
(throw (ex-info "User not found" {:user-id user-id :status :not-found}))
```

## Structure and Organization

### Bottom-Up Reading Order

Clojure files read bottom to top. Private helpers and building blocks live at the top
of the file; the public API lives at the bottom. Callees above callers. A reader
entering the file finds the high-level interface at the end, and can drill into
implementation details by scrolling up.

### Vertical Distance

Things that are related should be close together. If function A calls function B, they
should be near each other in the file. Don't make the reader jump around.

### Line Length

Keep lines under 80 characters. Long lines force horizontal scrolling, break side-by-side diffs, and make code harder to scan. If a line exceeds 80 characters, it's usually a sign that the expression is doing too much — extract a `let` binding, break the form across lines, or pull logic into a named function.

Exceptions where exceeding 80 characters is acceptable:
 - `:require` **statements** — a long namespace path or alias list
 - **URLs in comments** — you can't break a URL across lines
 - **String literals** — error messages, log messages, or other content where breaking mid-sentence hurts readability more than the long line does
 - **Test names** — descriptive `it` and `describe` strings should read as one sentence

The common thread: content that is inherently one unit where breaking it creates more noise than the long line does.

### File Length

Aim for files under 100 lines. Files over 200 lines almost certainly have multiple
responsibilities and should be split. Namespaces with hiccup templates (Clojure's
equivalent of HTML) are an exception — the template markup can push line counts up.
The 100-line target applies to the functional parts of the code (logic, handlers,
services, utilities).

### Screaming Architecture

The project's directory structure should scream what the application *does*, not what
framework it uses. A glance at the top-level namespaces should tell you the domain —
users, billing, campaigns — not the technical layers.

```
;; Bad — screams "I'm a web framework"
src/clj/myapp/controllers/
src/clj/myapp/models/
src/clj/myapp/services/

;; Good — screams "I manage users and billing"
src/clj/myapp/users/
src/clj/myapp/billing/
src/clj/myapp/campaigns/
src/clj/myapp/shared/
```

Group by feature or domain concept. Put the handler, service, validation, and specs
for "users" in the `users/` namespace tree — not scattered across four layer
directories.

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
you understand what varies and what doesn't.

## SOLID at a Glance

These principles originated in OOP but apply to functional code too:

- **Single Responsibility** — a namespace has one reason to change
- **Open/Closed** — extend behavior (multimethods, protocols) without modifying existing code
- **Interface Segregation** — don't force consumers to depend on functions they don't use; keep namespace APIs focused
- **Dependency Inversion** — depend on abstractions, not concrete implementations. Multimethods are usually the best fit; protocols work when you need polymorphism tied to a type

You don't need to cite these by name. Just follow them. If a namespace has two reasons
to change, split it. If you're modifying a `cond` every time you add a case, use a
multimethod.

## Agent-Friendly Code

These rules are good practice for humans too, but coding agents benefit from them
disproportionately. Agents pattern-match aggressively, read code literally, and
struggle with indirection. The cleaner and more consistent the code, the better
agents work with it.

### Consistency Over Preference

If the codebase does something one way, do it that way everywhere — even if you prefer
another style. Three different patterns for the same thing (three ways to fetch data,
three error response shapes, three component structures) means an agent will pick one
arbitrarily or mix them. One pattern, used everywhere, means the agent learns it once
and applies it correctly.

When in doubt, grep for precedent before introducing a new pattern.

### Colocate Context

If understanding a function requires reading 5 other namespaces, an agent has to load
all of them — and might not. Self-contained functions with minimal distant dependencies
are easier for agents to work with correctly. This reinforces screaming architecture:
keeping a feature's handler, validation, and specs in the same namespace tree means the
agent doesn't have to hunt.

### No Clever Code

Agents take code literally. Obscure macro tricks, overly dense one-liners, or "smart"
patterns that require deep Clojure expertise to parse will trip them up more than
they'd trip a human. If a junior developer would need to stop and think about it, an
agent will probably misread it.

Write obvious code. Save your cleverness for the architecture.

### Avoid Dynamic Dispatch

`eval`, `resolve`, `ns-resolve`, dynamically constructed var references — agents can't
statically trace any of these. They'll miss call sites, misunderstand what's being
invoked, and produce broken refactors. Multimethods and protocols are fine (they're
designed for extensible dispatch). Ad-hoc dynamic resolution is not.

### Organized Imports

Keep `:require` forms sorted alphabetically by namespace. Sorted requires are
scannable, diffable, and make duplicates obvious. An agent adding a new require to a
sorted list will place it correctly. An agent adding a require to an unsorted pile may
put it anywhere.

```clojure
(ns myapp.users.handler
  (:require [clojure.string :as str]
            [myapp.shared.validators :as validators]
            [myapp.users.service :as service]
            [ring.util.response :as response]))
```
