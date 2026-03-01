# [repo-name] — Rig Instructions

You are working on **[repo-name]**, [repo-description]>. Read these instructions before starting any work.

---

## Crew Directory

You are not working alone. These specialists are available on this rig. When you hit a
question that falls in someone else's specialty, reach out — don't guess.

| Name | Specialty | When to reach out |
|------|-----------|-------------------|
| **PM** | Rig coordinator, domain knowledge, project context | Scope questions, "where does this go?", "what's the current state of X?", project history |
| **Picasso** | UI/UX, components, styling, layout, interactivity | Design decisions, component structure, animation approach, "should this be a modal or inline?" |
| **Penny** | Security, bugs, dead code, test coverage | Security questions, testing approach, "is this path covered?", "is this input validated?" |
| **Bob** | Clean code, naming, structure, refactoring | Naming advice, structure questions, "is this clean enough?", pair refactoring |

### How to Reach Out

See `/docs/collaboration.md` for communication commands (`gt mail`, `gt nudge`, bead
comments) and guidance on when to ask vs. when to file a bead.

---

## Standards

This project has documented standards. Read the relevant ones before doing work in
that area.

| Doc | Covers |
|-----|--------|
| `/docs/testing-standards.md` | Testing philosophy, structure, what to test |
| `/docs/testing-setup.md` | React + Node testing conventions (Vitest, RTL) |
| `/docs/security-checklist.md` | Security audit guide |
| `/docs/validation-boundaries.md` | Frontend/backend validation contract |
| `/docs/clean-code-standards.md` | Naming, functions, structure, SOLID |
| `/docs/design-direction.md` | Visual direction, anti-patterns, inspiration |
| `/docs/domain.md` | Project architecture, conventions, current state |

You don't need to read all of them — just the ones relevant to your current bead.

---

## Conventions

- **Stack:** React + Node (no TypeScript)
- **ESM throughout** — no CommonJS
- **CSS Modules** for all component styling (`.module.css` co-located with components)
- **Named exports** for all components and hooks (no default exports)
- **Shared validation** lives in `shared/` — frontend and backend use the same rules
- **Tests** live in `tests/` mirroring `src/` and `server/`

See `/docs/domain.md` for the full list of conventions and project structure.
