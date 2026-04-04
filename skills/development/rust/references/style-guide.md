# Rust Style Guide Reference

Detailed style rules for writing clear, idiomatic Rust code.

## Formatting

Use `rustfmt` (via `cargo fmt`) for all formatting. Do NOT fight the formatter.

Recommended `rustfmt.toml`:

```toml
edition = "2024"
max_width = 100
use_field_init_shorthand = true
```

## Ownership and Borrowing Patterns

### Function Parameters

| Accept | When |
|--------|------|
| `&str` | Reading a string (not owning) |
| `String` | Taking ownership (storing in struct) |
| `&[T]` | Reading a slice |
| `Vec<T>` | Taking ownership of a vector |
| `impl AsRef<str>` | Accepting both `&str` and `String` |
| `impl Into<String>` | Will store, caller picks allocation |
| `&T` | Reading any borrowed type |
| `&mut T` | Mutating in place |
| `T` | Small `Copy` types or consuming ownership |

### Return Types

| Return | When |
|--------|------|
| `T` | Small types, owned values |
| `&T` | Borrowing from `self` |
| `Option<T>` | Value might not exist |
| `Result<T, E>` | Operation might fail |
| `impl Iterator` | Lazy sequence |
| `impl Future` | Async operation |
| `Box<dyn Trait>` | Dynamic dispatch (trait object) |

## Struct Design

### Field Visibility

```rust
// Public struct with private fields (controlled access)
pub struct Config {
    host: String,
    port: u16,
}

impl Config {
    pub fn new(host: impl Into<String>, port: u16) -> Self {
        Self { host: host.into(), port }
    }

    pub fn host(&self) -> &str { &self.host }
    pub fn port(&self) -> u16 { self.port }
}

// Data transfer object (all public, derive everything)
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct UserDto {
    pub id: u64,
    pub name: String,
    pub email: String,
}
```

### Field Init Shorthand

```rust
// Good: shorthand when variable matches field name
let name = "Alice".to_string();
let user = User { name, age: 30 };

// Good: struct update syntax
let updated = User { age: 31, ..user };
```

### Derive Order

Follow a consistent derive order:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Default, Serialize, Deserialize)]
```

Order: `Debug` → `Clone` → `Copy` → `PartialEq` → `Eq` → `PartialOrd` → `Ord` → `Hash` → `Default` → `Serialize` → `Deserialize`

## Enums

### Use Enums for State Machines

```rust
#[derive(Debug, Clone, PartialEq)]
enum ConnectionState {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session_id: String },
    Error { message: String, retryable: bool },
}
```

### Exhaustive Matching

ALWAYS handle all variants. Use `_` wildcard only when you genuinely don't care about future variants:

```rust
match state {
    ConnectionState::Disconnected => connect(),
    ConnectionState::Connecting { attempt } if attempt > 3 => give_up(),
    ConnectionState::Connecting { .. } => wait(),
    ConnectionState::Connected { session_id } => use_session(&session_id),
    ConnectionState::Error { retryable: true, .. } => retry(),
    ConnectionState::Error { message, .. } => log_error(&message),
}
```

## Generics and Trait Bounds

### Where Clauses

Use `where` for complex bounds:

```rust
// Simple: inline
fn print<T: Display>(item: T) { /* ... */ }

// Complex: where clause
fn process<T, E>(input: T) -> Result<(), E>
where
    T: AsRef<str> + Send + Sync,
    E: From<std::io::Error> + Display,
{
    // ...
}
```

### Impl Trait

```rust
// In parameters: accept anything implementing the trait
fn read_data(reader: impl Read) -> Result<Vec<u8>> { /* ... */ }

// In return: hide concrete type
fn create_filter() -> impl Fn(&str) -> bool { /* ... */ }
```

## Lifetime Annotations

### When Required

```rust
// Returning a reference borrowed from input
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}

// Struct holding references
struct Parser<'input> {
    source: &'input str,
    position: usize,
}

// Multiple lifetimes when they differ
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

### Elision Rules

The compiler infers lifetimes in most cases. Only annotate when:
1. A function returns a reference and has multiple reference parameters
2. A struct holds a reference
3. The compiler asks you to

## Closures

### Capture Rules

```rust
let name = String::from("Alice");

// Borrow: closure borrows name
let greet = || println!("Hello, {name}");

// Mutable borrow: closure mutably borrows list
let mut list = vec![];
let mut add = || list.push(1);

// Move: closure takes ownership
let print_name = move || println!("{name}");
// name is no longer available here
```

### Closure Types

| Trait | Called | Captures |
|-------|--------|----------|
| `Fn` | Multiple times | `&self` (shared borrow) |
| `FnMut` | Multiple times | `&mut self` (mutable borrow) |
| `FnOnce` | Once | `self` (ownership) |

## Iterator Patterns

Prefer iterator chains over manual loops:

```rust
// Good: expressive, composable
let active_names: Vec<String> = users
    .iter()
    .filter(|u| u.is_active())
    .map(|u| u.name().to_string())
    .collect();

// Avoid: manual loop when iterator is clearer
let mut active_names = Vec::new();
for u in &users {
    if u.is_active() {
        active_names.push(u.name().to_string());
    }
}
```

### Common Iterator Methods

| Method | Purpose |
|--------|---------|
| `map` | Transform each element |
| `filter` | Keep matching elements |
| `filter_map` | Filter and transform in one step |
| `flat_map` | Map and flatten nested iterators |
| `fold` | Reduce to a single value |
| `any` / `all` | Boolean check on elements |
| `find` | First matching element |
| `enumerate` | Add index to each element |
| `zip` | Pair two iterators |
| `chain` | Concatenate iterators |
| `take` / `skip` | Limit or offset elements |
| `collect` | Consume into a collection |

## Import Organization

Three groups separated by blank lines:

```rust
// 1. Standard library
use std::collections::HashMap;
use std::sync::Arc;

// 2. External crates
use dioxus::prelude::*;
use serde::{Deserialize, Serialize};
use tokio::sync::RwLock;

// 3. Internal modules
use crate::models::User;
use super::helpers;
```

Rules:
- Group `use` within each section alphabetically
- Use `{}` for multiple items from the same module
- Prefer explicit imports over glob imports (except `prelude::*`)
- Re-export key types from `lib.rs` for clean public API

## Code Organization Within Files

1. Module-level documentation (`//!`)
2. Imports (`use`)
3. Constants and statics
4. Type definitions (structs, enums, type aliases)
5. Trait definitions
6. Trait implementations (`impl Trait for Type`)
7. Inherent implementations (`impl Type`)
8. Free functions
9. Tests (`#[cfg(test)] mod tests`)

## Documentation Comments

```rust
/// Creates a new [`Config`] from environment variables.
///
/// # Errors
///
/// Returns [`ConfigError::Missing`] if `DATABASE_URL` is not set.
///
/// # Examples
///
/// ```
/// let config = Config::from_env()?;
/// assert!(!config.database_url().is_empty());
/// ```
pub fn from_env() -> Result<Config, ConfigError> { /* ... */ }
```

Rules:
- Use `///` for public items, `//!` for module-level docs
- First line is a summary sentence (imperative mood)
- Document `# Errors`, `# Panics`, `# Safety` sections when applicable
- Include runnable examples in `# Examples`
- Link to related types with [`TypeName`]

## Sources

- [Rust Style Guide](https://doc.rust-lang.org/nightly/style-guide/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/)
- [The Rust Programming Language](https://doc.rust-lang.org/stable/book/)
