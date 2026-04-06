# Testing Setup — Rust (claw-code Workspace)

Stack-specific conventions for the `rust/` Cargo workspace.

## Framework

- Built-in `cargo test`
- `tokio::test` for async tests
- `mock-anthropic-service` for deterministic API mocking
- Mock parity harness for end-to-end CLI behavior
- `insta` (if used) or `assert_cmd` for CLI output testing
- `tempfile` for temporary workspaces in tests

## File Organization

Tests live next to the code:

- Unit tests: `src/lib.rs` or `src/foo.rs` with `#[cfg(test)] mod tests { ... }`
- Integration tests: `tests/` directory at crate root or workspace `tests/`
- Parity harness: `crates/rusty-claude-cli/tests/mock_parity_harness.rs`

Example workspace test layout mirrors the crate structure.

## Mocking

- Prefer real in-memory implementations over mocks when possible
- Use the `mock-anthropic-service` crate for API parity
- `#[cfg(test)]` feature flags for test-only behavior
- `tempfile::TempDir` for file-system isolation

## Running Tests

See `testing-standards.md` for commands.

Ensure `cargo test --workspace` passes cleanly and is part of CI.

All parity scenarios in `mock_parity_scenarios.json` must remain green.

No `#[ignore]` tests allowed on main.