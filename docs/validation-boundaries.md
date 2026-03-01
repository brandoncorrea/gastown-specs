# Validation Boundaries

This document defines how input validation is structured across the frontend and backend. Both stacks (Clojure+ClojureScript and React+Node) share a language between frontend and backend. This is an advantage — use it.

## The Three Layers

### Layer 1: Shared Validations

Validation logic that applies on **both** sides of the wire. Since frontend and backend share a language, these validations are written once and used in both contexts.

Shared validations cover:
- Field presence (required vs optional)
- Type checks (string, number, boolean, date)
- Format validation (email shape, phone pattern, URL structure)
- Length and range constraints (min/max length, numeric bounds)
- Enum membership (status must be one of these values)
- Cross-field rules (end date must be after start date)

Shared validations live in a common location that both frontend and backend can import. Do not duplicate these — a single source of truth means a rule change applies everywhere.

### Layer 2: Frontend Validations

The frontend uses shared validations to provide **immediate user feedback**. This is primarily a UX concern, but it is also a practical security concern — thousands of invalid requests that could have been caught client-side waste server resources and can be used as a vector for abuse.

The frontend is responsible for:
- Running shared validations on form input before submission
- Displaying error messages next to the relevant fields
- Preventing submission of obviously invalid data (saving the user a round-trip and the server from unnecessary load)
- Guiding the user toward correct input
- Checking uniqueness against locally available data when the frontend already has a snapshot (e.g., user-owned entities already loaded into state)

The frontend is **not** responsible for:
- Enforcing business rules the user shouldn't know about
- Checking database-dependent constraints it doesn't have data for (uniqueness checks against the full database belong on the backend)
- Building complete entities — it submits only what the user provided

### Layer 3: Backend Validations

The backend is the **source of truth**. It runs shared validations again and adds server-only concerns. The backend trusts no external input — not from clients, not from external APIs, not from webhooks. Anything crossing a trust boundary is validated.

The backend is responsible for:
- Re-running all shared validations on incoming data
- Uniqueness checks (email already taken, duplicate record)
- Authorization checks (does this user have permission to create this?)
- Referential integrity (does the referenced parent record exist?)
- Database constraints (foreign keys, unique indexes)
- Building the complete entity from user input + server-controlled fields
- Validating data received from external services the same way it validates client data

## Type Coercion

Validation should do its best to coerce invalid types or formats to the correct type before rejecting them. A string `"3"` can become the number `3`. A date string in a common format can be parsed.

Keep coercion light and reasonable. The goal is to be forgiving of minor formatting differences, not to write an exhaustive format-guessing engine. If the input is close, coerce it. If it's nonsensical, reject it.

## The Entity Construction Rule

**The frontend submits user input. The backend builds the entity.**

This is the most important boundary in the system. Here's what it looks like:

### Wrong — frontend submits an entity

```
Frontend sends:
{
  id: "auto-generated-uuid",
  title: "My Post",
  body: "Hello world",
  authorId: "user-123",
  status: "published",
  createdAt: "2025-01-15T10:00:00Z",
  updatedAt: "2025-01-15T10:00:00Z"
}

Backend does:
db.save(payload)
```

An attacker changes `authorId` to someone else's ID, sets `status` to `"published"` bypassing review, or adds `role: "admin"` and the backend saves it all.

### Right — frontend submits input, backend builds entity

```
Frontend sends:
{
  title: "My Post",
  body: "Hello world"
}

Backend does:
1. Validate title and body (shared validations + length limits)
2. Coerce types if needed (trim strings, parse numbers)
3. Build the entity:
   {
     id: generateId(),
     title: validated.title,
     body: validated.body,
     authorId: authenticatedUser.id,
     status: "draft",
     createdAt: now(),
     updatedAt: now()
   }
4. Save the constructed entity
```

The user only controls what they should control. The backend controls everything else.

### What the Frontend Should Submit

Only fields the user directly provided or selected:
- Text input values
- Selected options from dropdowns
- Checkbox/toggle states
- File uploads

### What the Backend Should Add

- Record IDs
- Timestamps (created, updated)
- Ownership (author, creator — derived from the authenticated session)
- Status/workflow state (initial status, not user-chosen)
- Computed fields (totals, hashes, derived values)
- Anything the user should not be able to influence

## Updates and Edits

The same principle applies to updates. The frontend sends the fields the user changed. The backend:

1. Loads the existing entity
2. Validates the submitted changes (including coercion)
3. Applies only the allowed fields (whitelist, not blacklist)
4. Sets `updatedAt`, modified-by, etc.
5. Saves

Never merge `req.body` directly into an existing record. Whitelist the fields you accept.

## Error Responses

When validation fails on the backend, return structured errors that the frontend can map to specific fields:

```
{
  errors: {
    title: "Title must be between 1 and 200 characters",
    body: "Body is required"
  }
}
```

Use the same error message format and keys that the shared validations use, so the frontend can display backend errors the same way it displays client-side errors.
