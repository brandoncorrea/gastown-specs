# Clean Code Standards — Rust (claw-code Workspace)

Clean code is a UX problem for other developers, coding agents, and our future selves.

These rules are for the entire `rust/` Cargo workspace (9 crates under `crates/`). Any rule can be broken — but only with intention and a comment explaining why.

## Naming

- **Variables/functions** — `snake_case`, descriptive verbs: `calculate_context_window`, not `calc` or `do_thing`
- **Types** — `PascalCase` (structs, enums, traits)
- **Constants** — `SCREAMING_SNAKE_CASE`
- **Avoid abbreviations** unless universally understood in Rust (`ctx`, `cfg`, `id` are fine; `usr`, `mgr` are not)
- **Avoid generic names** — `data`, `info`, `item`, `result`, `temp` almost always have better names

If you need a comment to explain a name, the name is wrong.

**Rust conventions win** — follow `rustfmt` and `clippy` (`cargo fmt`, `cargo clippy --all-targets -- -D warnings`).

## Functions & Modules

- **Do one thing** — if you can extract another meaningful function, do it
- **Keep them short** — aim for < 30 lines; > 50 lines almost always violates single responsibility
- **Limit parameters** — 0-3 ideal; 4+ → use a struct or builder
- **No flag parameters** — split into separate functions (`execute_with_sandbox`, `execute_readonly`)
- **Minimize nesting** — max 2-3 levels; use early returns and guard clauses
- **Extract complex conditions** into well-named functions or `const` values
- **No side effects** unless the name makes it obvious (`execute_bash_and_capture_output`)

## Error Handling

- Prefer `Result<T, E>` over `unwrap`/`expect` in library code
- Use `thiserror` + `anyhow` appropriately (`thiserror` for public errors, `anyhow` for internal)
- Propagate errors with `?` — never swallow them silently
- `main` in `rusty-claude-cli` is allowed more liberal error handling

## No Unnecessary Ceremony

- Use `cargo fmt` religiously
- Prefer `iter()` methods and functional style when clearer
- Use `serde` derive macros instead of manual serialization
- Prefer `clap` derive for CLI parsing
- Use `tracing` or `log` macros instead of `println!` in library crates

## Rust-Specific Idioms

- Ownership and borrowing are part of the API — document when clones are expensive
- Prefer `&str` over `String` in function signatures when possible
- Use `Cow<'a, str>` when you sometimes need owned data
- Enums for error states and state machines (see `PermissionMode`, tool execution results)
- `#[derive(Debug, Clone, PartialEq, Eq)]` on public types unless there's a reason not to
- `#[non_exhaustive]` on public enums/structs that may grow

## Structure and Organization

### Screaming Architecture
The crate layout already screams the domain:
- `api/` — external services
- `runtime/` — core conversation & session logic
- `tools/` — built-in agent tools
- `commands/` — CLI and slash-command surface
- `plugins/`, `telemetry/`, etc.

Keep it that way. New functionality belongs in the crate that owns that domain.

### File Length
- Aim for < 400 lines per file (excluding tests)
- `lib.rs` should be a short public API overview

### The Boy Scout Rule
Leave every file cleaner than you found it (`cargo fmt`, fix one `clippy` lint, improve one name).

## DRY — But Not Prematurely

Extract shared code after you see the **exact same pattern three times**. Rust's type system makes premature abstraction painful — wait until the shape is clear.

## Agent-Friendly Code

- Be explicit — agents pattern-match literally
- Prefer static dispatch and clear call sites
- One clear public item per module when possible
- Keep `lib.rs` files as clean public APIs
- Document any intentional `unsafe` blocks with safety comments
- Use `#[cfg(test)]` modules for test-only helpers

Follow `clippy` and `rustfmt` — they are the source of truth for style in this workspace.