# Rust Naming Conventions Reference

Comprehensive naming rules following RFC 430 and Rust API Guidelines.

## Master Table

| Element | Convention | Good | Bad |
|---------|-----------|------|-----|
| Crate | `snake_case` | `my_app` | `myApp`, `my-app-rs` |
| Module | `snake_case` | `user_auth` | `userAuth`, `UserAuth` |
| Type / Struct | `UpperCamelCase` | `HttpClient` | `HTTP_Client`, `httpClient` |
| Enum | `UpperCamelCase` | `Color` | `COLORS` |
| Enum variant | `UpperCamelCase` | `Color::DarkRed` | `Color::DARK_RED` |
| Trait | `UpperCamelCase` | `Iterator`, `Display` | `iterator` |
| Function | `snake_case` | `get_users` | `getUsers`, `GetUsers` |
| Method | `snake_case` | `is_empty` | `isEmpty` |
| Local variable | `snake_case` | `user_count` | `userCount` |
| Struct field | `snake_case` | `first_name` | `firstName` |
| Constant | `SCREAMING_SNAKE_CASE` | `MAX_SIZE` | `MaxSize`, `max_size` |
| Static | `SCREAMING_SNAKE_CASE` | `GLOBAL_POOL` | `global_pool` |
| Type parameter | `UpperCamelCase` (short) | `T`, `E`, `Item` | `t`, `type` |
| Lifetime | `'lowercase` (short) | `'a`, `'ctx` | `'A`, `'CTX` |
| Feature flag | `snake_case` | `full_stack` | `fullStack` |
| Macro | `snake_case!` | `vec!`, `println!` | `Vec!` |

## Conversion Methods

The prefix encodes the cost and ownership semantics:

| Prefix | Cost | Input | Output | Example |
|--------|------|-------|--------|---------|
| `as_` | Free | `&self` | `&T` | `str::as_bytes()` → `&[u8]` |
| `to_` | Expensive | `&self` | `T` (new) | `str::to_string()` → `String` |
| `into_` | Variable | `self` | `T` (consume) | `String::into_bytes()` → `Vec<u8>` |
| `from_` | Variable | `T` | `Self` | `String::from_utf8()` |
| `try_from_` | Fallible | `T` | `Result<Self>` | `u32::try_from(i64)` |
| `try_into_` | Fallible | `self` | `Result<T>` | `i64::try_into::<u32>()` |

## Constructor Patterns

```rust
// Primary constructor
impl Config {
    pub fn new(path: &str) -> Self { /* ... */ }
}

// Variant constructors
impl Config {
    pub fn with_defaults() -> Self { /* ... */ }
    pub fn from_env() -> Result<Self> { /* ... */ }
}

// Builder pattern for complex construction
impl ServerBuilder {
    pub fn new() -> Self { /* ... */ }
    pub fn port(mut self, port: u16) -> Self { self.port = port; self }
    pub fn host(mut self, host: String) -> Self { self.host = host; self }
    pub fn build(self) -> Result<Server> { /* ... */ }
}
```

## Getter and Setter Patterns

```rust
impl User {
    // Getter: NO get_ prefix
    pub fn name(&self) -> &str { &self.name }
    pub fn age(&self) -> u32 { self.age }

    // Boolean: use is_/has_/can_ prefix
    pub fn is_active(&self) -> bool { self.active }
    pub fn has_permission(&self, p: &str) -> bool { /* ... */ }

    // Setter: use set_ prefix
    pub fn set_name(&mut self, name: String) { self.name = name; }
}
```

## Iterator Method Names

```rust
impl Collection {
    // Shared reference iterator
    pub fn iter(&self) -> Iter<'_, Item> { /* ... */ }

    // Mutable reference iterator
    pub fn iter_mut(&mut self) -> IterMut<'_, Item> { /* ... */ }

    // Consuming iterator (impl IntoIterator)
    pub fn into_iter(self) -> IntoIter<Item> { /* ... */ }
}
```

## Error Naming

```rust
// Error enum: <Domain>Error suffix
#[derive(Debug, thiserror::Error)]
pub enum ParseError {
    #[error("invalid syntax at line {line}")]
    InvalidSyntax { line: usize },
    #[error("unexpected token: {0}")]
    UnexpectedToken(String),
    #[error("io error")]
    Io(#[from] std::io::Error),
}

// Result alias (in library crates)
pub type Result<T> = std::result::Result<T, ParseError>;
```

## Trait Naming

Traits describe capabilities -- name them as adjectives or with `-able`/`-ible` suffixes:

| Pattern | Example | When |
|---------|---------|------|
| Adjective | `Clone`, `Send`, `Sync` | Describes a property |
| Verb-able | `Readable`, `Serializable` | Describes a capability |
| Noun-er | `Iterator`, `Formatter` | Describes a role |
| Verb | `Read`, `Write`, `Display` | Standard library style |

## Module Naming

```rust
// Good: descriptive, singular
mod auth;
mod config;
mod handler;
mod model;

// Bad: plural, generic, stuttering
mod utils;      // what kind of utils?
mod helpers;    // too vague
mod models;     // use singular
mod auth_auth;  // stuttering
```

## Feature Flag Naming

```rust
[features]
default = ["web"]
web = ["dioxus/web"]
server = ["dioxus/server"]
desktop = ["dioxus/desktop"]
full = ["web", "server"]     # NOT "fullstack" or "all"
```

## Dioxus-Specific Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Component | `UpperCamelCase` | `UserCard`, `NavBar` |
| Hook | `use_snake_case` | `use_auth`, `use_theme` |
| Signal | `snake_case` | `let count = use_signal(...)` |
| GlobalSignal | `SCREAMING_SNAKE_CASE` | `static THEME: GlobalSignal<...>` |
| Server function | `snake_case` | `get_users`, `create_post` |
| Route enum | `Route` with `UpperCamelCase` variants | `Route::Home`, `Route::UserDetail` |
| CSS class | `kebab-case` (in RSX strings) | `class: "user-card"` |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `get_name()` | `name()` -- Rust omits `get_` |
| `HTTPClient` | `HttpClient` -- only first letter capitalized in acronyms for CamelCase |
| `my-crate-rs` | `my_crate` -- no `-rs` suffix, use underscores |
| `utils` module | Name by purpose: `string_helpers`, `date_format` |
| `ALL_CAPS` for enum variants | `UpperCamelCase`: `Status::Active` not `STATUS_ACTIVE` |
| `impl new()` without `pub` | Constructors should be `pub fn new()` |
| `_unused` prefix to silence warning | Remove the binding or use it; `_` for intentional ignore |
| Lifetime `'lifetime` | Keep short: `'a`, `'b`, or semantic: `'ctx`, `'db` |

## Sources

- [Rust API Guidelines -- Naming](https://rust-lang.github.io/api-guidelines/naming.html)
- [RFC 430 -- Naming Conventions](https://github.com/rust-lang/rfcs/blob/master/text/0430-finalizing-naming-conventions.md)
- [Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/)
