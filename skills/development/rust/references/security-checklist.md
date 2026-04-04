# Rust Security Checklist Reference

Security review checklist organized by domain.

## Input Validation

- [ ] All user input is validated at system boundaries
- [ ] String lengths are bounded before processing
- [ ] Numeric inputs are range-checked
- [ ] File paths are sanitized (no path traversal: `../`)
- [ ] URL inputs are validated against allowlists
- [ ] Content-Type headers are checked before parsing

```rust
// Good: validate at boundary
fn create_user(input: &CreateUserRequest) -> Result<User> {
    ensure!(input.name.len() <= 100, "name too long");
    ensure!(!input.email.is_empty(), "email required");
    ensure!(input.email.contains('@'), "invalid email format");
    // ...
}
```

## SQL Injection

- [ ] ALL database queries use parameterized statements
- [ ] NEVER construct SQL with `format!()` or string concatenation
- [ ] Use `sqlx::query!` or `diesel` for compile-time checked queries

```rust
// Good: parameterized query
let user = sqlx::query_as!(
    User,
    "SELECT * FROM users WHERE id = $1",
    user_id
)
.fetch_one(&pool)
.await?;

// DANGEROUS: string interpolation in SQL
let query = format!("SELECT * FROM users WHERE name = '{name}'"); // SQL INJECTION!
```

## Command Injection

- [ ] NEVER pass user input through a shell
- [ ] Use `std::process::Command` with separate arguments
- [ ] Validate command arguments against allowlists

```rust
// Good: separate arguments
let output = Command::new("git")
    .arg("log")
    .arg("--oneline")
    .arg("-n")
    .arg(count.to_string())
    .output()?;

// DANGEROUS: shell execution with user input
let output = Command::new("sh")
    .arg("-c")
    .arg(format!("git log --oneline -n {user_input}")) // COMMAND INJECTION!
    .output()?;
```

## Cross-Site Scripting (XSS)

- [ ] Dioxus RSX auto-escapes string interpolation (safe by default)
- [ ] NEVER use `dangerous_inner_html` with user-provided content
- [ ] Sanitize any HTML rendered from external sources
- [ ] Set Content-Security-Policy headers

```rust
// Safe: Dioxus auto-escapes
rsx! { p { "{user_input}" } }  // HTML entities escaped

// DANGEROUS: raw HTML injection
rsx! { div { dangerous_inner_html: "{user_input}" } }  // XSS!
```

## Authentication and Sessions

- [ ] Passwords hashed with `argon2` or `bcrypt` (NEVER plain text, MD5, SHA)
- [ ] Session tokens generated with `rand::rngs::OsRng` (CSPRNG)
- [ ] Tokens have expiration times
- [ ] CSRF protection on state-changing endpoints
- [ ] Rate limiting on authentication endpoints

```rust
use argon2::{Argon2, PasswordHash, PasswordHasher, PasswordVerifier};
use argon2::password_hash::SaltString;
use rand::rngs::OsRng;

fn hash_password(password: &str) -> Result<String> {
    let salt = SaltString::generate(&mut OsRng);
    let hash = Argon2::default()
        .hash_password(password.as_bytes(), &salt)?
        .to_string();
    Ok(hash)
}

fn verify_password(password: &str, hash: &str) -> Result<bool> {
    let parsed = PasswordHash::new(hash)?;
    Ok(Argon2::default().verify_password(password.as_bytes(), &parsed).is_ok())
}
```

## Secrets Management

- [ ] NO hardcoded secrets, API keys, or passwords in source code
- [ ] Secrets loaded from environment variables or secret managers
- [ ] `.env` files in `.gitignore`
- [ ] Secrets not logged (redact in `Debug` implementations)
- [ ] Use constant-time comparison for secret comparison

```rust
// Good: from environment
let api_key = std::env::var("API_KEY")
    .context("API_KEY environment variable not set")?;

// Good: constant-time comparison
use subtle::ConstantTimeEq;
fn verify_token(provided: &[u8], expected: &[u8]) -> bool {
    provided.ct_eq(expected).into()
}

// DANGEROUS: hardcoded secret
const API_KEY: &str = "sk-1234567890"; // NEVER DO THIS!
```

## Cryptography

- [ ] Use `ring`, `rustls`, or `rcgen` -- NEVER implement custom crypto
- [ ] Use `OsRng` for all random generation (NEVER `thread_rng` for security)
- [ ] TLS 1.2+ for all network communication
- [ ] Certificates validated (do NOT disable verification)

## Unsafe Code

- [ ] Minimize `unsafe` blocks -- prefer safe alternatives
- [ ] Every `unsafe` block has a `// SAFETY:` comment
- [ ] Run `cargo geiger` to audit unsafe usage
- [ ] Unsafe code is covered by tests including edge cases
- [ ] Consider `#![forbid(unsafe_code)]` for application crates

```rust
// Good: documented safety invariant
// SAFETY: we've verified that index < slice.len() on the line above
unsafe { *slice.get_unchecked(index) }

// Bad: undocumented unsafe
unsafe { *slice.get_unchecked(index) }
```

## Dependencies

- [ ] Run `cargo audit` regularly (in CI)
- [ ] Run `cargo deny check` for license and advisory checks
- [ ] Pin dependency versions in `Cargo.lock` (committed for binaries)
- [ ] Review new dependencies before adding (check crate popularity, maintenance)
- [ ] Minimize dependency tree -- fewer deps = smaller attack surface

```bash
cargo audit                    # check for known vulnerabilities
cargo deny check               # license + advisory + ban checks
cargo geiger                   # count unsafe usage in deps
cargo tree                     # visualize dependency tree
```

## Network Security

- [ ] Use `rustls` over `openssl` when possible (memory-safe TLS)
- [ ] Set timeouts on all HTTP clients
- [ ] Validate SSL certificates (do NOT use `danger_accept_invalid_certs`)
- [ ] Implement request size limits
- [ ] Use HTTPS for all external communication

```rust
// Good: reqwest with timeouts
let client = reqwest::Client::builder()
    .timeout(Duration::from_secs(30))
    .connect_timeout(Duration::from_secs(5))
    .build()?;
```

## Dioxus-Specific Security

- [ ] Server functions validate all inputs (client can send anything)
- [ ] Sensitive logic gated with `#[cfg(feature = "server")]`
- [ ] No secrets in client-side code (WASM is inspectable)
- [ ] Server functions implement authentication checks
- [ ] Rate limiting on server function endpoints

```rust
#[server]
async fn delete_user(user_id: u64) -> Result<(), ServerFnError> {
    // MUST verify authorization -- client can call any server function
    let session = get_session().await?;
    if !session.is_admin() {
        return Err(ServerFnError::new("unauthorized"));
    }
    db::delete_user(user_id).await
        .map_err(|e| ServerFnError::new(e.to_string()))
}
```

## CI Security Pipeline

```bash
cargo fmt --check                      # consistent formatting
cargo clippy -- -D warnings            # static analysis
cargo test                             # all tests pass
cargo audit                            # no known vulnerabilities
cargo deny check                       # license compliance
cargo geiger                           # unsafe audit
```

## Sources

- [Rust Secure Coding Guidelines](https://anssi-fr.github.io/rust-guide/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [RustSec Advisory Database](https://rustsec.org/)
- [Cargo Deny](https://embarkstudios.github.io/cargo-deny/)
