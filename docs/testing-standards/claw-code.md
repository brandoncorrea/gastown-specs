# Testing Standards — Rust (claw-code)

Reference guide for test quality. Read before writing any test in the `rust/` workspace.

## Test Naming

Tests are specifications. Use descriptive names:

**Good:** `"rejects write outside workspace when permission_mode is read_only"`
**Bad:** `"test file write"`

Pattern: `<expected behavior> when <condition>`

## Test Structure

Every test follows Arrange → Act → Assert.

Use `#[test]`, `#[tokio::test]`, or integration tests in `tests/`.

Extract common setup into helper functions or the mock parity harness.

## One Behavior Per Test

Each test verifies **one logical behavior**. Multiple assertions are fine if they verify the same behavior.

## Test Behavior, Not Implementation

- Test public APIs and crate contracts
- Refactoring internals should not break tests
- Prefer testing through the `claw` binary (via mock parity harness) for end-to-end behavior

**Exception:** Unit tests for complex pure functions (permission checks, context window calculation, etc.).

## Test Independence

- No test depends on another test’s state
- Use `#[serial]` only when absolutely necessary (rare)
- Reset mock state or use fresh temp directories

## Implicit Coverage Is Real Coverage

Not every module needs its own `.rs` test file. If a helper is exercised by higher-level tests (especially parity harness), it’s covered.

Promote complex shared logic to its own testable unit only when it has a clear contract and multiple consumers.

## Dead Code

Delete it. Delete its tests. Verify the suite still passes.

## What to Test

- Happy path
- Edge cases (empty, max size, boundary values)
- Error cases (permission denied, invalid input, network failures)
- Security boundaries (see security-checklist.md)
- Parity scenarios (streaming, file ops, tool calls, bash sandboxing)

## What NOT to Test

- Third-party crate behavior
- Implementation details that can change
- Dead code

## Backend / Crate Testing

- **Unit tests** — inside `#[cfg(test)]` modules or `tests/` in each crate
- **Integration / Parity tests** — `crates/rusty-claude-cli/tests/mock_parity_harness.rs` + `scripts/run_mock_parity_harness.sh`
- Use the mock Anthropic service for deterministic API testing

## Assertion Style

- Prefer `assert_eq!`, `assert!`, `insta` for snapshots where output is complex
- Be precise when the exact value is part of the contract

## Coverage Philosophy

Focus on critical paths: permission enforcement, tool execution, API streaming, config loading, error recovery.

## Running Tests

```bash
cd rust
cargo test --workspace          # all tests
cargo test --package rusty-claude-cli --test mock_parity_harness
cargo test -p tools             # single crate
cargo test -- --quiet           # less output
```

CI must pass `cargo test --workspace` and `cargo fmt --all --check`.
Follow Red → Green → Refactor. Commit after each green step.
