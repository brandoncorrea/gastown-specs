# Security Checklist

Audit guide for Go backend services — HTTP APIs, RPC services, and the workers behind
them. Work through this checklist for every feature or package you review.

The threats are a malicious or careless **client**, a compromised or buggy **service**
on either side of a call — something calling us, or something we call — and our own
**supply chain**. A backend is the source of truth: a validation bug here is not
cosmetic, it is corrupted state that every other consumer then trusts.

## The Core Principle

**Never trust external input.** Anything that crosses a trust boundary is validated
before use. A backend has three such boundaries — and the third is the one people
forget:

1. **Requests from clients** — browsers, mobile apps, CLIs, anything with a user behind it
2. **Requests from other services** — webhooks, callbacks, internal or partner APIs
3. **Responses from services we call** — every body we read back from an outbound call
   is as untrusted as an inbound request

See `/docs/validation-boundaries.md` for how validation is layered.

## 1. Input Validation

### The Caller/Service Contract
- [ ] Callers submit **only what they control** — the input for an operation, not a
      fully-formed entity
- [ ] The service **builds the entity**, adding the fields it controls: IDs, owner,
      status, version, computed values, timestamps
- [ ] Every inbound field is validated, regardless of who the caller claims to be
- [ ] Responses from other services are validated the same way requests are — they
      can carry unexpected shapes, missing fields, or hostile values

### What to Check
- [ ] No handler decodes a request body straight into a stored type, and no decoded
      struct is persisted as-is
- [ ] Decode errors are handled. A body that failed to parse is a `400`, not a
      zero-valued struct that continues down the happy path
- [ ] Path and query parameters are parsed with error handling — no `Must*` function
      ever sees caller input
- [ ] Generated types are not mistaken for validation. Decoding into types generated
      from an OpenAPI spec or schema checks JSON shape only; patterns, enums, ranges,
      and required fields must be enforced explicitly
- [ ] Enum-like strings are checked against the allowed values
- [ ] Numeric fields have range checks (quantities, amounts, page sizes)
- [ ] Strings and collections have size limits
- [ ] Nested objects are validated — not just the top-level fields. Optional pointer
      fields are nil-checked before they're dereferenced
- [ ] Times parse, and ranges are ordered (end after start)
- [ ] Request bodies are size-limited with `http.MaxBytesReader`
- [ ] Response bodies we read are size-limited with `io.LimitReader`
- [ ] File uploads are checked for size and content type (sniffed with
      `http.DetectContentType`, not trusted from the header or extension)

### Red Flag
A handler that does the equivalent of `json.Unmarshal(body, &entity); store.Save(entity)`
— or that fills server-controlled fields from the request — is a **critical finding**.
A caller can set any field it likes: someone else's `owner_id`, `role: "admin"`,
`price: 0`, a `status` it isn't allowed to enter.

## 2. Authentication

### Inbound
- [ ] Every protected request is authenticated on every request — not cached by
      caller, not skipped for "internal" paths
- [ ] Bearer tokens (JWTs): the signature is verified with the algorithm pinned —
      `alg: none` and HMAC-with-the-public-key are rejected. `exp` is enforced, `iss`
      matches the configured issuer, and `aud` matches **this** service. A token minted
      for another service must not work here
- [ ] The signing key or JWKS location comes from configuration, never from the token
- [ ] API keys and shared secrets are compared with `crypto/subtle.ConstantTimeCompare`
      and stored hashed, not in plain text
- [ ] Webhook and callback requests are authenticated — a verified signature, not
      "it came to the secret URL"
- [ ] Authentication failures return `401` with a generic message — no hints about
      which check failed or whether an account exists

### If the Service Talks to Browsers
- [ ] Session cookies set `HttpOnly`, `Secure`, and `SameSite`
- [ ] Logging out destroys the session server-side
- [ ] State-changing requests authenticated by cookie are protected against CSRF
      with `http.CrossOriginProtection`
- [ ] CORS allows specific origins, never `*` with credentials
- [ ] Passwords are hashed with a slow KDF (`bcrypt`, `argon2id`); reset tokens are
      single-use and time-limited

### Outbound (we are the client)
- [ ] Credentials for each downstream service are scoped to that service. A token
      obtained for one service is never sent to another
- [ ] A caller's inbound token is not forwarded to third parties
- [ ] Credentials come from the environment or a secret manager, never from source
- [ ] Development-only auth (fake token sources, auth-disabled modes) cannot be
      selected by accident in a production configuration

## 3. Authorization

- [ ] Every operation checks that the caller has the **permission that operation
      requires** — a scope, a role, or a policy decision — not just that they are
      authenticated
- [ ] The caller's identity comes from the verified credential (the token's `sub`, the
      session) — never from a field in the request body
- [ ] Callers can only read or modify entities they own or are entitled to. Changing an
      ID in the path or body does not grant access to someone else's entity
- [ ] Admin and internal endpoints are not reachable with ordinary client credentials
- [ ] Operations over a collection check permission for each item, not just the first

### Red Flag
A route registered without the auth middleware is open to the internet. Route tests
must prove that an unauthenticated request never reaches the handler — see "Testing
HTTP Services" in `/docs/testing-standards.md`.

## 4. Injection and SSRF

- [ ] **SSRF:** any URL we request that came from input or from another service
      (webhook targets, callback URLs, image fetches) is checked before calling it: the
      scheme is one we allow, the host is not loopback/link-local/private unless the
      deployment explicitly allows it, and redirects are not followed blindly
- [ ] URLs are built with `net/url` (`JoinPath`, `PathEscape`, `url.Values`), never by
      concatenating caller-supplied strings
- [ ] Database queries use parameters — never string concatenation or `fmt.Sprintf`
- [ ] Caller input is never passed to a shell. `exec.Command` takes arguments
      separately; no `sh -c` with interpolated input
- [ ] File paths built from input are opened through `os.Root`, so `..` and symlinks
      cannot escape the directory
- [ ] HTML is rendered with `html/template`, never `text/template`
- [ ] Log calls pass caller input as structured attributes, never as part of the
      format string or message
- [ ] JSON responses set `Content-Type: application/json`

## 5. Data Exposure

- [ ] Responses contain only the fields the API defines for that operation — response
      types are separate from stored types, so a new column doesn't leak by default
- [ ] Error responses do not leak internal paths, wrapped error chains, SQL, or stack
      traces. Log the detail; return the API's error shape with a plain message
- [ ] Tokens, `Authorization` headers, cookies, passwords, and client secrets are never
      logged — including by request/response logging middleware
- [ ] Secrets and endpoints come from environment variables or a secret manager, not
      from source
- [ ] `.env` is in `.gitignore`; any example env file contains no real values

## 6. Availability and Abuse

- [ ] The HTTP server sets `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, and
      `IdleTimeout` — an unbounded server is a slowloris target
- [ ] Every outbound client has a timeout, and every outbound call carries the
      request's `context`
- [ ] A slow or dead dependency cannot stall requests indefinitely or exhaust goroutines
- [ ] Retries are bounded and back off
- [ ] Operations that fan out (notifying subscribers, fetching many resources) are
      bounded in concurrency and in count
- [ ] Expensive or abusable endpoints (login, search, sign-up) are rate-limited
- [ ] List endpoints paginate with a maximum page size
- [ ] Shutdown drains in-flight requests within a deadline

## 7. Concurrency

In Go, a data race is a memory-safety bug, not just a logic bug.

- [ ] State shared between request goroutines — including in-memory stores and caches
      — is guarded by a mutex or confined to one goroutine
- [ ] Check-then-act sequences on shared state are atomic (read version, compare,
      write) — in memory with a lock, in the database with a transaction or a
      conditional update
- [ ] `go test -race ./...` is clean
- [ ] Every goroutine has an owner and a stop condition; none outlive their request
      without a reason

## 8. Dependencies and Supply Chain

- [ ] The `go` directive in `go.mod` names the latest Go release, and dependencies are
      on their latest versions (`go get -u ./...`)
- [ ] `govulncheck ./...` reports no reachable vulnerabilities
- [ ] `go.sum` is committed; `go mod verify` passes
- [ ] `go mod tidy` leaves no diff — unused dependencies are removed
- [ ] Generated code is produced only by the project's generate step, from a pinned
      source (spec, schema) and a pinned generator version. Files marked
      `// Code generated ... DO NOT EDIT.` are never hand-edited
- [ ] CI actions and container base images are pinned

## Severity Levels

When reporting findings, classify them:

- **Critical** — Exploitable now with no authentication or trivial effort, or able to
  corrupt stored state. Examples: an unauthenticated route, a decoded body persisted
  as-is, a caller-supplied owner or role being trusted, secrets in source.
- **High** — Exploitable by an authenticated caller or requires some knowledge.
  Examples: missing permission or audience checks, modifying another user's entity by
  ID, SSRF through a caller-supplied URL, a panic reachable from caller input, a data
  race on a store.
- **Medium** — Defense-in-depth gaps. Examples: missing body size limits, missing
  timeouts, overly detailed error messages, unbounded fan-out, no rate limiting.
- **Low** — Best practice improvements. Examples: unpinned CI actions, an unused
  dependency, a credential scope broader than needed.
