# Security Checklist — claw-code Rust CLI

Audit guide for every feature, crate, or tool you review in the `rust/` workspace.

## The Core Principle

**Never trust external input.** Not from CLI arguments, not from environment variables, not from `.claw.json` configs, not from prompt text, not from tool responses, not from plugins, not from the Anthropic API. Any data that crosses a trust boundary must be validated and/or sandboxed before use. The Rust binary is the source of truth — everything else is untrusted until proven otherwise.

## 1. Input Validation

### The CLI / Runtime Contract
- [ ] The `rusty-claude-cli` binary only accepts raw user input (args, flags, REPL text, config files)
- [ ] `runtime` and `api` crates build proper types from input, adding server-controlled fields (timestamps, session IDs, ownership, computed values)
- [ ] Every crate validates **all** input independently
- [ ] Validation logic is shared via `serde` derives + custom `Deserialize` impls where possible
- [ ] Backend validations extend shared ones with Rust-specific concerns (ownership, lifetimes, permission checks)

### What to Check
- [ ] No crate blindly deserializes user input into a struct that reaches `tools` or `bash` execution
- [ ] All string inputs are bounded (max length) and sanitized where appropriate
- [ ] Numeric inputs have range validation
- [ ] Enum fields use Rust enums with exhaustive matching (no stringly-typed fallbacks)
- [ ] Vectors and collections have enforced size limits
- [ ] File paths are canonicalized and checked against workspace boundaries (`PermissionEnforcer`)
- [ ] Tool arguments (especially `bash`, `write_file`, `edit_file`) are validated by `PermissionEnforcer::check_*`
- [ ] API responses from Anthropic (or mock) are validated before use — never assume schema

### Red Flag
If you see `serde_json::from_str(&user_input)` directly feeding into tool execution or file writes without `PermissionEnforcer`, that is a **critical finding**. An attacker can inject arbitrary commands or paths.

## 2. Authentication & Secrets

- [ ] `ANTHROPIC_API_KEY` (and OAuth tokens) are never logged, never included in prompts, never persisted in plain text
- [ ] Secrets come from environment or secure keyring — never from source or untrusted config
- [ ] OAuth flow in `api` crate uses proper state and PKCE
- [ ] Token expiration and refresh are handled securely
- [ ] `logout` actually clears credentials
- [ ] Authentication failures return generic messages (no user enumeration)

## 3. Authorization & Permissions

- [ ] Every tool execution (bash, file ops, plugins, MCP, LSP) goes through `PermissionEnforcer`
- [ ] Users cannot bypass permissions by changing flags, config, or tool arguments
- [ ] `dangerously-skip-permissions` flag is clearly documented as dangerous and requires explicit opt-in
- [ ] Permission checks happen **before** any side effect (in `runtime` and `tools`)
- [ ] Bulk or recursive operations (glob, grep, directory edits) verify every item

### Red Flag
Any code path that reaches `std::process::Command` or file I/O without going through `PermissionEnforcer::check_bash()` / `check_file_write()` / etc. is a critical vulnerability.

## 4. Injection & Command Execution

- [ ] Bash tool uses sandboxing (`unshare`, timeouts, read-only gating where possible)
- [ ] No direct `std::process::Command::arg(user_input)` without validation
- [ ] File operations prevent path traversal, symlinks, and out-of-workspace writes
- [ ] All external calls (Anthropic API, plugins, MCP) use typed clients — never shelling out to curl
- [ ] HTML/Markdown rendering escapes user content where displayed

## 5. Data Exposure & Secrets Management

- [ ] No sensitive data (API keys, session tokens, full prompts) is logged or included in telemetry unless explicitly allowed
- [ ] Error messages in production never leak stack traces, file paths, or internal state (use `anyhow` or custom error types with `#[non_exhaustive]`)
- [ ] Config files (`.claw.json`) never store secrets
- [ ] `.env` and secrets are in `.gitignore`
- [ ] Telemetry crate only sends anonymized usage data

## 6. Rate Limiting & Resource Abuse

- [ ] API calls to Anthropic respect rate limits and context windows (preflight in `api` crate)
- [ ] Expensive tools (large file reads, heavy grep, recursive operations) have size/time limits
- [ ] REPL input and prompt size are bounded

## 7. Dependencies & Supply Chain

- [ ] No known vulnerabilities (`cargo audit`)
- [ ] Dependencies are pinned in `Cargo.toml` (workspace and per-crate)
- [ ] Unused dependencies are removed
- [ ] Only trusted crates from crates.io (review `Cargo.lock` changes)

## Severity Levels

- **Critical** — Exploitable now with trivial effort (e.g., missing `PermissionEnforcer` check on `bash`, raw deserialization into file write, hardcoded secrets).
- **High** — Authenticated bypass or missing validation on sensitive tools.
- **Medium** — Defense-in-depth gaps (missing bounds, verbose errors, unpinned deps).
- **Low** — Best-practice improvements (`clippy` lints, better error messages, etc.).

Run this checklist for every new tool, command, or crate change.