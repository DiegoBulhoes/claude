# Rust Project Layout Reference

Project structure examples by type and complexity.

## Decision: Ask First

NEVER over-structure. Ask the developer about project type (web app, fullstack, library, CLI) and preferred patterns before creating the layout.

## Dioxus Web App (Client-Only)

```
<PROJECT>/
├── Cargo.toml
├── Dioxus.toml
├── assets/
│   ├── favicon.ico
│   ├── main.css
│   └── images/
├── src/
│   ├── main.rs              # launch(App), Route enum
│   ├── components/
│   │   ├── mod.rs
│   │   ├── header.rs
│   │   ├── footer.rs
│   │   └── ui/              # Generic UI primitives
│   │       ├── mod.rs
│   │       ├── button.rs
│   │       └── modal.rs
│   ├── views/
│   │   ├── mod.rs
│   │   ├── home.rs
│   │   └── about.rs
│   ├── hooks/
│   │   ├── mod.rs
│   │   └── use_auth.rs
│   └── models/
│       ├── mod.rs
│       └── user.rs
└── tests/
    └── integration_test.rs
```

### main.rs Pattern

```rust
use dioxus::prelude::*;

mod components;
mod hooks;
mod models;
mod views;

use views::*;

#[derive(Routable, Clone, PartialEq)]
enum Route {
    #[layout(Layout)]
    #[route("/")]
    Home {},
    #[route("/about")]
    About {},
    #[route("/:..segments")]
    NotFound { segments: Vec<String> },
}

fn main() {
    dioxus::launch(App);
}

#[component]
fn App() -> Element {
    rsx! { Router::<Route> {} }
}

#[component]
fn Layout() -> Element {
    rsx! {
        components::Header {}
        main { Outlet::<Route> {} }
        components::Footer {}
    }
}
```

## Dioxus Fullstack App

```
<PROJECT>/
├── Cargo.toml
├── Dioxus.toml
├── assets/
│   └── main.css
├── src/
│   ├── main.rs
│   ├── components/
│   │   └── mod.rs
│   ├── views/
│   │   └── mod.rs
│   ├── hooks/
│   │   └── mod.rs
│   ├── models/
│   │   ├── mod.rs
│   │   └── user.rs          # Shared types (client + server)
│   └── server/
│       ├── mod.rs
│       ├── auth.rs           # #[server] functions for auth
│       └── api.rs            # #[server] functions for data
├── migrations/               # Database migrations (sqlx)
└── tests/
    ├── server_test.rs
    └── integration_test.rs
```

### Cargo.toml for Fullstack

```toml
[package]
name = "<PROJECT_NAME>"
version = "0.1.0"
edition = "2024"

[dependencies]
dioxus = { version = "0.6", features = ["fullstack"] }
serde = { version = "1", features = ["derive"] }

[target.'cfg(feature = "server")'.dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }
tokio = { version = "1", features = ["full"] }

[features]
default = []
web = ["dioxus/web"]
server = ["dioxus/server"]
```

### Server Function Pattern

```rust
// src/server/api.rs
use dioxus::prelude::*;
use crate::models::User;

#[server]
pub async fn get_users() -> Result<Vec<User>, ServerFnError> {
    let pool = get_db_pool().await?;
    let users = sqlx::query_as!(User, "SELECT id, name, email FROM users")
        .fetch_all(&pool)
        .await
        .map_err(|e| ServerFnError::new(e.to_string()))?;
    Ok(users)
}

#[server]
pub async fn create_user(name: String, email: String) -> Result<User, ServerFnError> {
    let pool = get_db_pool().await?;
    let user = sqlx::query_as!(
        User,
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
        name, email
    )
    .fetch_one(&pool)
    .await
    .map_err(|e| ServerFnError::new(e.to_string()))?;
    Ok(user)
}
```

## Rust Library

```
<PROJECT>/
├── Cargo.toml
├── src/
│   ├── lib.rs               # Public API surface
│   ├── types.rs              # Public types
│   ├── error.rs              # Error types (thiserror)
│   └── internal/             # Private implementation
│       ├── mod.rs
│       └── parser.rs
├── examples/
│   └── basic.rs              # Runnable examples
├── benches/
│   └── benchmark.rs          # Criterion benchmarks
└── tests/
    └── integration_test.rs
```

Rules for libraries:
- Keep the public API surface minimal
- Re-export key types from `lib.rs`
- Use `pub(crate)` for internal-only items
- Provide runnable examples in `examples/`
- Define clear error types with `thiserror`

## CLI Tool

```
<PROJECT>/
├── Cargo.toml
├── src/
│   ├── main.rs              # Parse args, call run()
│   ├── cli.rs               # Clap argument definitions
│   ├── commands/
│   │   ├── mod.rs
│   │   ├── init.rs
│   │   └── build.rs
│   └── config.rs
└── tests/
    └── cli_test.rs
```

### main.rs for CLI

```rust
use anyhow::Result;

mod cli;
mod commands;
mod config;

fn main() -> Result<()> {
    let args = cli::parse();
    match args.command {
        cli::Command::Init(opts) => commands::init::run(opts),
        cli::Command::Build(opts) => commands::build::run(opts),
    }
}
```

## Workspace (Multi-Crate)

```
<PROJECT>/
├── Cargo.toml               # [workspace] definition
├── crates/
│   ├── <APP>/
│   │   ├── Cargo.toml
│   │   └── src/
│   ├── <CORE_LIB>/
│   │   ├── Cargo.toml
│   │   └── src/
│   └── <SHARED_TYPES>/
│       ├── Cargo.toml
│       └── src/
└── Dioxus.toml               # If using Dioxus
```

### Workspace Cargo.toml

```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.dependencies]
dioxus = { version = "0.6" }
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

## Small Project (Flat Layout)

For simple scripts, small tools, or prototypes:

```
<PROJECT>/
├── Cargo.toml
├── Dioxus.toml
├── assets/
│   └── main.css
└── src/
    └── main.rs
```

This is perfectly fine. Not every project needs `components/` and `views/`.

## Essential Configuration Files

### Dioxus.toml

```toml
[application]
name = "<APP_NAME>"
default_platform = "web"

[web.app]
title = "<APP_TITLE>"

[web.watcher]
reload_html = true
watch_path = ["src", "assets"]

[web.resource.dev]
style = ["/assets/main.css"]
```

### .gitignore

```
# Build artifacts
/target
/dist

# Dioxus
/out

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store

# Environment
.env
.env.local
```

### rustfmt.toml

```toml
edition = "2024"
max_width = 100
use_field_init_shorthand = true
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Over-structuring small projects | Flat `src/main.rs` is fine for prototypes |
| Giant `main.rs` with everything | Split into modules early for web apps |
| Server logic accessible from client | Gate with `#[cfg(feature = "server")]` |
| Missing `mod.rs` in directories | Every directory needs a `mod.rs` or use `module_name.rs` pattern |
| Shared types not serializable | Add `Serialize + Deserialize` to models used across client/server |
| Assets not in `assets/` directory | Dioxus serves from `assets/` -- put static files there |

## Sources

- [Dioxus Documentation](https://dioxuslabs.com/learn/0.6/) -- Project structure and setup
- [Cargo Book](https://doc.rust-lang.org/cargo/) -- Workspace and package layout
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) -- Library structure
