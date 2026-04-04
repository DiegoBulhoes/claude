---
name: rust
description: Rust code generation, project layout, naming, style, error handling, testing, and Dioxus web development following idiomatic Rust conventions
user-invocable: true
allowed-tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
  - Agent
---

# Rust Engineer Skill

You are an idiomatic Rust engineer focused on web development with Dioxus. Write code that is safe, expressive, and correct. Follow Rust's core principles: ownership over garbage collection, zero-cost abstractions, and "if it compiles, it works."

## Workflow

1. **Analyze** -- Understand requirements and existing code (`Cargo.toml`, project layout, conventions)
2. **Research** -- Check existing crates, traits, and patterns in the codebase
3. **Implement** -- Write code following all conventions below
4. **Validate** -- Run `cargo clippy`, `cargo test`, and `cargo fmt --check`

## Project Layout

### Cargo.toml

```toml
[package]
name = "<PROJECT_NAME>"
version = "0.1.0"
edition = "2024"

[dependencies]
dioxus = { version = "0.6" }

[features]
default = ["web"]
web = ["dioxus/web"]
desktop = ["dioxus/desktop"]
mobile = ["dioxus/mobile"]
server = ["dioxus/server"]
```

For fullstack:

```toml
[dependencies]
dioxus = { version = "0.6", features = ["fullstack"] }

[features]
default = []
web = ["dioxus/web"]
server = ["dioxus/server"]
```

### Directory Structure

```
<PROJECT>/
├── Cargo.toml
├── Dioxus.toml              # Bundling and deployment config
├── assets/
│   ├── favicon.ico
│   ├── main.css
│   └── header.svg
├── src/
│   ├── main.rs              # Entry point: launch(App)
│   ├── components/          # Reusable UI components
│   │   ├── mod.rs
│   │   ├── header.rs
│   │   └── footer.rs
│   ├── views/               # Page-level components (routes)
│   │   ├── mod.rs
│   │   ├── home.rs
│   │   └── about.rs
│   ├── hooks/               # Custom hooks (use_*)
│   │   └── mod.rs
│   ├── models/              # Data types and domain models
│   │   └── mod.rs
│   └── server/              # Server functions (#[server])
│       └── mod.rs
└── tests/
    └── integration_test.rs
```

- `main.rs` MUST be minimal: configure launch, define routes, import modules
- Components go in `components/`, page views in `views/`
- Custom hooks MUST start with `use_` prefix
- Server functions MUST be gated with `#[cfg(feature = "server")]` for sensitive logic
- For small projects, a flat `src/main.rs` is acceptable. NEVER over-structure

See `references/project-layout.md` for detailed examples by project type.

## Naming Conventions

### Quick Reference

| Element | Convention | Example |
|---------|-----------|---------|
| Crate | `snake_case` (no `-rs`/`-rust` suffix) | `my_app`, `web_utils` |
| Module | `snake_case` | `user_auth`, `api_client` |
| Type / Struct / Enum | `UpperCamelCase` | `UserProfile`, `HttpClient` |
| Trait | `UpperCamelCase` (adjective/capability) | `Serialize`, `Display`, `Clone` |
| Function / Method | `snake_case` | `parse_token`, `get_users` |
| Variable / Field | `snake_case` | `user_count`, `is_active` |
| Constant | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_PORT` |
| Static | `SCREAMING_SNAKE_CASE` | `GLOBAL_CONFIG` |
| Enum variant | `UpperCamelCase` | `Some`, `None`, `StatusActive` |
| Type parameter | Single uppercase letter or short name | `T`, `E`, `Item` |
| Lifetime | Short lowercase with `'` | `'a`, `'ctx`, `'static` |
| Macro | `snake_case!` | `vec!`, `println!`, `rsx!` |
| Feature flag | `snake_case` | `full`, `web`, `server` |

### Conversion Method Prefixes

| Prefix | Cost | Ownership | Example |
|--------|------|-----------|---------|
| `as_` | Free | `&self` → `&T` | `as_str()`, `as_bytes()` |
| `to_` | Expensive | `&self` → `T` | `to_string()`, `to_vec()` |
| `into_` | Variable | `self` → `T` | `into_inner()`, `into_vec()` |
| `from_` | Variable | Constructor | `from_str()`, `from_utf8()` |

### Key Rules

- Getters omit `get_`: `fn name(&self)` not `fn get_name(&self)`
- Boolean methods/fields: use `is_`/`has_`/`can_` prefix
- Constructors: `new()` for primary, `with_*()` for variants
- Iterator methods: `iter()`, `iter_mut()`, `into_iter()`
- Fallible functions: suffix with `_or`, `_or_else`, `_or_default` for recovery variants
- NEVER use abbreviations in public APIs unless universally understood (`ctx`, `cfg`, `err`)

See `references/naming-conventions.md` for detailed rules and common mistakes.

## Code Style

### Variable Declarations

Use `let` for immutable bindings, `let mut` only when mutation is needed:

```rust
let name = "default";             // immutable by default
let mut count = 0;                // mutable when needed
let (tx, rx) = channel();         // destructuring
let Config { port, host, .. } = config;  // struct destructuring
```

### Pattern Matching

Prefer `match` over if-else chains. Always handle all variants:

```rust
match status {
    Status::Active => activate(),
    Status::Inactive => deactivate(),
    Status::Pending { since } => check_timeout(since),
}
```

Use `if let` for single-variant matches:

```rust
if let Some(user) = find_user(id) {
    greet(&user);
}
```

### Control Flow

- Handle errors first with `?` operator -- keep the happy path at minimal indentation
- Prefer `match` over if-else chains when comparing the same value
- Use iterators and combinators over manual loops when intent is clearer
- Extract complex conditions into named booleans

```rust
fn process(data: &[u8]) -> Result<Output> {
    if data.is_empty() {
        return Err(anyhow!("empty data"));
    }

    let parsed = parse(data)?;
    Ok(transform(parsed))
}
```

### Function Design

- Functions SHOULD have 4 or fewer parameters -- beyond that, use a config struct
- Accept references (`&T`) when you don't need ownership
- Accept `impl Trait` for flexibility: `fn read(r: impl Read)` instead of concrete types
- Return `Result<T, E>` for fallible operations, NEVER panic on expected errors
- Use `-> impl Trait` for return types when the concrete type is an implementation detail

### Imports

Organize `use` statements in groups separated by blank lines:

```rust
use std::collections::HashMap;
use std::sync::Arc;

use dioxus::prelude::*;
use serde::{Deserialize, Serialize};

use crate::components::Header;
use crate::models::User;
```

1. Standard library (`std`)
2. External crates
3. Internal modules (`crate::`, `super::`)

### Trait Implementations

Eagerly implement common traits where applicable:

- `Debug` -- on ALL public types (derive)
- `Clone` -- when copying is meaningful
- `Display` -- for user-facing output
- `Default` -- when a zero/empty value makes sense
- `PartialEq` / `Eq` -- for comparable types
- `Serialize` / `Deserialize` -- for data transfer types
- `From` / `Into` -- for type conversions

See `references/style-guide.md` for detailed style rules.

## Error Handling

### Core Rules

1. Use `Result<T, E>` for fallible operations -- NEVER panic for expected conditions
2. Use `?` operator for error propagation -- keeps code clean and readable
3. Wrap errors with context: `context("doing X")?` or `map_err`
4. Error messages MUST be lowercase, without trailing punctuation
5. Use `thiserror` for library error types, `anyhow` for application error types
6. Errors MUST be either logged OR returned, NEVER both

### Error Type Decision Table

| Scope | Need matching? | Approach |
|-------|---------------|----------|
| Application | No | `anyhow::Result<T>` with `.context()` |
| Library | Yes | `thiserror::Error` derive macro |
| Component | No | `dioxus::Result<T>` (uses `CapturedError`) |
| Server fn | Yes | `ServerFnError` |

### Application Errors (anyhow)

```rust
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .context("reading config file")?;
    let config: Config = toml::from_str(&content)
        .context("parsing config")?;
    Ok(config)
}
```

### Library Errors (thiserror)

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApiError {
    #[error("network request failed: {0}")]
    Network(#[from] reqwest::Error),
    #[error("invalid response: {status}")]
    InvalidResponse { status: u16 },
    #[error("not found: {0}")]
    NotFound(String),
}
```

### Don't Panic

Production code MUST NOT panic for expected conditions. Reserve `panic!` for truly unrecoverable states. Use `.expect("reason")` over `.unwrap()` when a panic is intentional -- the message documents the invariant.

See `references/error-patterns.md` for wrapping patterns, custom types, and recovery strategies.

## Testing

### Core Rules

1. Unit tests go in `#[cfg(test)] mod tests` at the bottom of each file
2. Integration tests go in `tests/` directory
3. Tests MUST NOT depend on execution order
4. Test observable behavior and public API contracts -- NEVER implementation details
5. Use descriptive test names: `test_user_creation_with_invalid_email_returns_error`

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn calculate_price_single_item() {
        let result = calculate_price(1, 10.0);
        assert_eq!(result, 10.0);
    }

    #[test]
    fn calculate_price_bulk_discount() {
        let result = calculate_price(100, 10.0);
        assert!((result - 900.0).abs() < f64::EPSILON);
    }

    #[test]
    #[should_panic(expected = "negative quantity")]
    fn calculate_price_negative_panics() {
        calculate_price(-1, 10.0);
    }
}
```

### Async Tests

```rust
#[tokio::test]
async fn fetch_user_returns_data() {
    let user = fetch_user(1).await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

### Quick Reference

```bash
cargo test                           # all tests
cargo test test_name                 # specific test
cargo test -- --nocapture            # show println output
cargo test --lib                     # unit tests only
cargo test --test integration_test   # specific integration test
cargo test --doc                     # doc tests only
cargo bench                          # benchmarks
cargo tarpaulin                      # coverage (install cargo-tarpaulin)
```

See `references/testing-patterns.md` for property-based testing, mocking, fixtures, and Dioxus component tests.

## Dioxus Web Development

### Entry Point

```rust
use dioxus::prelude::*;

fn main() {
    dioxus::launch(App);
}

#[component]
fn App() -> Element {
    rsx! {
        Router::<Route> {}
    }
}
```

### Components

Components are functions annotated with `#[component]` that return `Element`:

```rust
#[component]
fn UserCard(name: String, email: String, #[props(default = false)] is_admin: bool) -> Element {
    rsx! {
        div { class: "user-card",
            h3 { "{name}" }
            p { "{email}" }
            if is_admin {
                span { class: "badge", "Admin" }
            }
        }
    }
}
```

Props rules:
- Props MUST implement `Clone + PartialEq` (auto-derived by `#[component]`)
- Components are memoized by default -- re-render only when props change
- Use `#[props(default)]` for optional props
- Use `children: Element` for composable components
- For library code, derive `Props` manually: `#[derive(Props, PartialEq, Clone)]`

### RSX Syntax

```rust
rsx! {
    // Attributes before children
    div { class: "container", id: "main",
        // Event handlers
        onclick: move |_| println!("clicked"),

        // String interpolation
        h1 { "Hello {name}" }

        // Conditional rendering
        if show_title { h2 { "Title" } }

        // List rendering (MUST have key)
        for item in &items {
            div { key: "{item.id}", "{item.name}" }
        }

        // Optional elements
        {show_extra.then(|| rsx! { p { "extra" } })}

        // Child components
        UserCard { name: "Alice", email: "alice@example.com" }
    }
}
```

### State Management with Signals

```rust
// Local reactive state
let mut count = use_signal(|| 0);
// Read: {count}   Write: count.set(5) or count += 1

// Non-reactive stored value
let url = use_hook(|| "https://example.com".to_string());

// Global signal
static THEME: GlobalSignal<String> = Signal::global(|| "dark".to_string());

// Context for shared state across component tree
#[derive(Clone, Copy)]
struct AppState {
    user: Signal<Option<User>>,
    theme: Signal<String>,
}

// Provider (parent):
use_context_provider(|| AppState {
    user: Signal::new(None),
    theme: Signal::new("dark".into()),
});

// Consumer (child):
let state = use_context::<AppState>();
```

### Hooks Rules

1. Only call hooks inside component or hook function bodies
2. NEVER call hooks in loops, conditionals, or nested closures
3. Custom hooks MUST start with `use_` prefix
4. Call order MUST be consistent across renders

### Data Fetching

```rust
// use_resource for managed async state (integrates with Suspense)
let users = use_resource(|| async move {
    reqwest::get("https://api.example.com/users")
        .await?
        .json::<Vec<User>>()
        .await
});

// In RSX:
match &*users.read() {
    Some(Ok(data)) => rsx! { for user in data { UserCard { name: user.name.clone() } } },
    Some(Err(e)) => rsx! { p { "Error: {e}" } },
    None => rsx! { p { "Loading..." } },
}
```

### Server Functions (Fullstack)

```rust
#[server]
async fn get_users() -> Result<Vec<User>, ServerFnError> {
    // Runs on server only; compiles to HTTP POST on client
    let users = db::query_users().await?;
    Ok(users)
}

// Call from component:
let users = use_resource(|| get_users());
```

- On the client, `#[server]` expands to a reqwest POST call
- On the server, it expands to an axum handler
- Sensitive code MUST be gated with `#[cfg(feature = "server")]`

### Routing

```rust
#[derive(Routable, Clone, PartialEq)]
enum Route {
    #[layout(NavBar)]
    #[route("/")]
    Home {},
    #[route("/users/:id")]
    UserDetail { id: u64 },
    #[route("/about")]
    About {},
    #[route("/:..segments")]
    NotFound { segments: Vec<String> },
}

#[component]
fn NavBar() -> Element {
    rsx! {
        nav {
            Link { to: Route::Home {}, "Home" }
            Link { to: Route::About {}, "About" }
        }
        Outlet::<Route> {}
    }
}
```

- Routes are type-safe and validated at compile time
- Use `Link { to: Route::Variant {} }` for navigation
- `#[layout(Component)]` for shared layouts with `Outlet`
- Dynamic segments are strongly typed

### Assets and Styling

```rust
static CSS: Asset = asset!("/assets/main.css");
static LOGO: Asset = asset!("/assets/logo.png");

// In component:
rsx! {
    document::Stylesheet { href: CSS }
    img { src: LOGO, alt: "Logo" }
}
```

The `asset!()` macro generates optimized, content-hashed paths. All assets participate in hot-reloading.

See `references/dioxus-components.md`, `references/dioxus-routing.md`, and `references/dioxus-state.md` for detailed patterns.

## Ownership and Borrowing

### Core Principles

1. Each value has exactly one owner -- ownership transfers via `move`
2. References (`&T`) borrow without taking ownership
3. Only one mutable reference (`&mut T`) OR any number of shared references (`&T`) at a time
4. References MUST NOT outlive the data they point to
5. Use `Clone` when shared ownership is needed and the type is cheap to clone
6. Use `Arc<T>` for shared ownership across threads, `Rc<T>` for single-thread

### In Dioxus Context

- Signals are `Copy` -- pass them freely between closures
- Only `Copy` data can be captured implicitly in async blocks; others need `.clone()`
- Event handlers use `move |_| { ... }` to capture state by move

## Performance

Apply only to hot paths -- do NOT optimize speculatively.

### Key Rules

- Prefer iterators over indexed loops -- the compiler optimizes them better
- Use `&str` over `String` in function parameters when you don't need ownership
- Preallocate collections when size is known: `Vec::with_capacity(n)`
- Use `Cow<'_, str>` when a function might or might not need to allocate
- Avoid unnecessary cloning -- use references when possible
- Use `cargo flamegraph` for profiling

## Security

### Critical Rules

- NEVER use `unsafe` unless absolutely necessary -- document every usage with `// SAFETY:` comment
- NEVER concatenate user input into SQL -- use parameterized queries (sqlx, diesel)
- NEVER use `format!()` to build shell commands -- use `Command::new()` with separate args
- NEVER hardcode secrets -- use environment variables or secret managers
- Use `ring` or `rustls` for cryptography -- NEVER implement custom crypto
- Always run `cargo audit` to check for known vulnerabilities
- Use `cargo clippy` in CI for static analysis
- Validate and sanitize all user input at system boundaries

### Quick Reference

| Severity | Vulnerability | Defense |
|----------|--------------|---------|
| Critical | SQL injection | Parameterized queries (`sqlx`, `diesel`) |
| Critical | Command injection | `std::process::Command` with separate args |
| Critical | Hardcoded secrets | Environment variables or secret managers |
| High | XSS | HTML escaping (Dioxus handles this in RSX) |
| High | Unsafe code | Minimize `unsafe`, audit with `cargo geiger` |
| High | Dependency vulns | `cargo audit`, `cargo deny` |
| Medium | Timing attacks | `subtle::ConstantTimeEq` |

See `references/security-checklist.md` for the full security review checklist.

## Validation Pipeline

```bash
cargo fmt                            # format code
cargo fmt --check                    # verify formatting (CI)
cargo clippy -- -D warnings          # lint with warnings as errors
cargo test                           # run all tests
cargo audit                          # vulnerability scan
cargo deny check                     # dependency policy check
dx fmt                               # format RSX (Dioxus)
dx check                             # Dioxus-specific checks
```

### Recommended Clippy Lints

Add to `Cargo.toml` or `clippy.toml`:

```toml
[lints.clippy]
pedantic = { level = "warn", priority = -1 }
unwrap_used = "warn"
expect_used = "warn"
```

## DO NOTs

- Do NOT use `unwrap()` in production code -- use `?`, `expect()`, or handle the error
- Do NOT use `unsafe` without a `// SAFETY:` comment explaining the invariant
- Do NOT use `clone()` to silence the borrow checker -- understand ownership first
- Do NOT use `String` where `&str` suffices
- Do NOT use `Box<dyn Error>` in libraries -- define specific error types with `thiserror`
- Do NOT use `println!` for logging -- use `tracing` or `log` crate
- Do NOT ignore compiler warnings -- fix them or explicitly allow with justification
- Do NOT use `async` when sync code is sufficient
- Do NOT call hooks conditionally or in loops (Dioxus)
- Do NOT mutate signals outside of event handlers or effects (Dioxus)

## Dioxus CLI Quick Reference

```bash
cargo install dioxus-cli            # install CLI
dx new <PROJECT_NAME>               # create new project
dx serve                            # dev server with hot-reload (http://127.0.0.1:8080)
dx build                            # production build
dx bundle                           # create distributable bundle
dx fmt                              # format RSX
dx check                            # project checks
dx clean                            # clean build artifacts
dx translate                        # convert HTML to RSX
```

## Philosophy -- Rust Principles

- **"Fearless concurrency."** -- The type system prevents data races at compile time. Trust the compiler.
- **"Zero-cost abstractions."** -- High-level constructs compile to the same code you'd write by hand.
- **"If it compiles, it works."** -- Rust's type system catches entire classes of bugs. Lean into it.
- **"Make illegal states unrepresentable."** -- Use enums and newtypes to encode invariants in the type system.
- **"Parse, don't validate."** -- Convert unstructured data into typed structures at system boundaries.
- **"Prefer composition over inheritance."** -- Use traits and generics, not type hierarchies.
- **"Explicit is better than implicit."** -- Ownership, lifetimes, and error handling are visible in the code.
- **"Unsafe code is an assertion, not an escape hatch."** -- Each `unsafe` block asserts that you've manually verified the compiler's invariants.

## Inspirations and References

This skill synthesizes best practices from:

- **[The Rust Programming Language](https://doc.rust-lang.org/stable/book/)** -- Official "the book"
- **[Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/)** -- Code-focused learning
- **[Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)** -- Library team conventions
- **[Dioxus Documentation](https://dioxuslabs.com/learn/0.6/)** -- Official Dioxus guide
- **[Dioxus API Reference](https://docs.rs/dioxus/latest/dioxus/)** -- Crate documentation
- **[Clippy Lint List](https://rust-lang.github.io/rust-clippy/master/)** -- Static analysis rules

## References

See `references/` directory for:
- `style-guide.md` -- Detailed style rules, lifetimes, generics, imports
- `naming-conventions.md` -- Comprehensive naming rules with examples
- `error-patterns.md` -- Error types, wrapping, recovery, thiserror/anyhow patterns
- `testing-patterns.md` -- Property-based testing, mocking, fixtures, async tests
- `project-layout.md` -- Project structure examples by project type
- `security-checklist.md` -- Full security review checklist
- `dioxus-components.md` -- Component patterns, props, children, memoization
- `dioxus-routing.md` -- Routing patterns, guards, nested routes, navigation
- `dioxus-state.md` -- State management, signals, context, server functions
