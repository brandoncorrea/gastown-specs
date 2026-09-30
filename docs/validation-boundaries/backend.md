# Validation Boundaries

This document defines how input validation is structured in a Go backend. For the
audit checklist, see `/docs/security-checklist.md`.

## Where the Boundaries Are

The service trusts nothing that arrives over the network. There are three places data
crosses into the system, and all three get the same treatment:

| Boundary | Examples |
|----------|----------|
| **Requests from clients** | `POST /orders`, `PATCH /orders/{id}` |
| **Requests from other services** | A payment provider's webhook, a partner API calling us |
| **Responses to our own calls** | The payment provider's charge result, another service's lookup |

The third is the one that gets forgotten. A response body from another service is
external input. It can be malformed, missing fields, or hostile, and it is validated
exactly like a request.

## The Three Layers

### Layer 1: Wire Validation

Does the payload conform to the API contract?

The API spec (OpenAPI, protobuf, or whatever the project uses) is the contract. Go
types generated from it are **not** validation. Decoding into a generated struct proves
the JSON had roughly the right shape and nothing more — it does not check required
fields, string patterns, enum membership, or numeric ranges. Those are checked
explicitly.

Wire validation covers:
- The body parses, and the parse error is handled
- Required fields are present (a nil pointer on a required field is a failure)
- Formats: UUIDs, RFC 3339 timestamps, URLs, emails
- Enum membership: statuses, currencies, units
- Ranges: quantities, amounts, lengths, page sizes
- Collection bounds: at least one item, at most the limit the spec or we impose

Wire validation is pure: no I/O, no clock, no store.

### Layer 2: Domain Validation

Does the payload make sense?

- Cross-field rules: end is after start, the discount doesn't exceed the subtotal
- Time rules: a reservation isn't in the past, a refund is within its window
- State rules: the requested transition is legal from the current state

Domain validation may read the clock and the current entity. It doesn't call out.

### Layer 3: Authorization and State

Is this caller allowed to do this, now?

- The credential carries the permission this operation requires, for this service
- The caller — identified by the verified credential, never by a body field — owns or
  is entitled to the entity it is trying to change
- The version (or ETag) presented is current
- Referenced entities exist

This layer needs the store and the authenticated identity. It runs last, so that cheap
rejections happen before expensive ones.

### Where It Lives in the Code

Validate **once, at the boundary**, and convert to domain types in the same step. This
is "parse, don't validate": the handler turns wire strings into `time.Time`,
`uuid.UUID`, and typed constants, and everything past it works with values that cannot
be malformed. The core never re-checks, because the types make the invalid states
unrepresentable.

```go
// Illustrative. The wire type: what the caller sent. Strings, nothing guaranteed.
type wireWindow struct {
	Start string `json:"start"`
	End   string `json:"end"`
}

// The domain type: cannot hold an unparsed or inverted window.
type Window struct {
	Start, End time.Time
}

// Parse validates and converts in one step. Past this call, a Window is valid.
func (wire wireWindow) Parse() (Window, error) {
	start, startErr := time.Parse(time.RFC3339Nano, wire.Start)
	end, endErr := time.Parse(time.RFC3339Nano, wire.End)
	if err := errors.Join(startErr, endErr); err != nil {
		return Window{}, err
	}
	if !end.After(start) {
		return Window{}, errors.New("end is not after start")
	}
	return Window{Start: start, End: end}, nil
}
```

- Validation functions return `error`. Use `errors.Join` to report every problem at
  once rather than the first one.
- Shared rules (what makes an address valid) are written once and used at every
  boundary — the address in a client's request and the address in a partner's response
  obey the same rule.

## Be Strict

**Reject off-contract input. Do not coerce it.**

An API defines its formats, and a backend's callers are programs — even when a person
is behind them, a frontend or client library sits between. Guessing what a malformed
timestamp or an out-of-range quantity "probably meant" is how two systems come to
disagree about the same record. Forgiveness toward a human typing belongs in the
frontend.

- Go's typed decoding already refuses a string where a number belongs. Don't work
  around it.
- Don't trim, lowercase, default, or clamp a caller's values into validity. If the
  domain defines a canonical form (a lowercased email), normalize explicitly, in one
  place, as a documented rule — not as a side effect of validation.
- Unknown fields are ignored and never stored — a typed struct drops them for free.
  (Use `Decoder.DisallowUnknownFields` if the API contract says to reject them.)
- When the contract says a field is optional, absent is valid. When it says required,
  absent is an error — not a zero value.

## The Entity Construction Rule

**The caller submits input. The service builds the entity.**

This is the most important boundary in the system.

### Wrong — the caller submits an entity

```
Caller sends:
{
  "id": "2f8343be-6482-4d1b-a474-16847e01af1e",
  "customer_id": "someone-else",
  "status": "paid",
  "total": 0,
  "version": 4,
  "items": [ ... ]
}

Service does:
json.Unmarshal(body, &order)
store.SaveOrder(order)
```

A caller claims someone else's `customer_id`, sets its own `total`, jumps straight to a
`status` it never earned, or bumps `version` to win a conflict — and the service saves
it all.

### Right — the caller submits input, the service builds the entity

```
Caller sends:
{
  "items": [ { "sku": "A-100", "quantity": 2 } ],
  "shipping_address": { ... }
}

Service does:
1. Validate and parse the request (Layers 1 and 2)
2. Authorize the caller (Layer 3)
3. Call any services it depends on, and validate their responses (Layers 1 and 2 again)
4. Build the entity:
     ID              ← generated by the service
     CustomerID      ← from the authenticated identity
     Items           ← validated.items
     Total           ← computed from catalog prices
     Status          ← derived by the service's rules
     ShippingAddress ← validated.shippingAddress
     Version         ← from the store
     CreatedAt       ← from the service's clock
5. Save the constructed entity
```

The caller only controls what it should control. The service and the systems it trusts
control everything else. The function that builds an entity should look like the list
above: every field assigned from a named source, one per line.

### What the Caller May Supply

- The choices that are genuinely theirs: what they're ordering, where it ships
- Their own client-side reference or idempotency key
- Filters, sort order, and page cursors for reads

### What the Service Supplies

- Entity IDs, versions, and timestamps of record
- Ownership — identity comes from the credential, never the body
- Status — derived by rule, not accepted as given
- Computed values — prices, totals, balances
- Anything the caller should not be able to influence

### In Go

- **Decode into a request type. Never into the stored type.** The request struct and
  the entity struct are different types even when their fields look alike today. This
  is what makes the typed struct a whitelist: a field that isn't on the request type
  cannot be submitted.
- **Never persist a decoded struct.** Construct the entity with a keyed literal, one
  field per line, so a reviewer can see where every value came from.
- **Encode from a response type, too.** A stored type serialized straight to the wire
  leaks every field anyone later adds to it.
- **Avoid `map[string]any`** for request or response bodies. It has no fields for the
  compiler, the reviewer, or a search to see.

## Updates

The same principle applies to updates. The caller sends the fields it may change. The
service:

1. Loads the existing entity
2. Validates the submitted changes
3. Authorizes — the caller may change this entity, and its version is current
4. Applies only the caller-controlled fields, by explicit assignment
5. Sets version and timestamps itself
6. Saves

Never overlay a decoded request onto an existing record. Assign the fields you accept.

## Error Responses

The shape of an error is dictated by the API's contract — whatever the spec defines,
or one shape the project has chosen (for example RFC 9457 problem details) and uses
everywhere.

- Status codes mean what HTTP says they mean: `400` for a malformed request, `401` for
  missing or bad credentials, `403` for a caller who may not do this, `404`, `409` for
  a stale version or conflict, `413` for an oversized body, `422` if the contract
  distinguishes well-formed-but-invalid, `429` for rate limits.
- Messages say what was wrong with the input (`"end is before start"`). They never
  include wrapped internal errors, SQL, file paths, or stack traces — log those.
- All responses go through one helper (`writeJSON`, `writeError`), so the content type,
  encoding, and error shape are decided in one place.
