# Rust Error Handling Patterns Reference

Error types, wrapping, recovery, and best practices.

## Error Strategy Decision

| Context | Crate | Pattern |
|---------|-------|---------|
| Application / binary | `anyhow` | `anyhow::Result<T>` with `.context()` |
| Library / public API | `thiserror` | Custom error enum with `#[derive(Error)]` |
| Dioxus component | built-in | `dioxus::Result<T>` (wraps `CapturedError`) |
| Server function | built-in | `Result<T, ServerFnError>` |
| Quick prototype | `anyhow` | `anyhow::Result<T>` everywhere |

## Application Errors with anyhow

```rust
use anyhow::{bail, ensure, Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .context("reading config file")?;

    let config: Config = toml::from_str(&content)
        .context("parsing config TOML")?;

    ensure!(config.port > 0, "port must be positive, got {}", config.port);

    if config.host.is_empty() {
        bail!("host cannot be empty");
    }

    Ok(config)
}
```

### anyhow Key Functions

| Function | Purpose |
|----------|---------|
| `context("msg")` | Add context to any error |
| `with_context(\|\| format!(...))` | Add lazy context (expensive messages) |
| `bail!("msg")` | Return early with error |
| `ensure!(cond, "msg")` | Assert condition, bail if false |
| `anyhow!("msg")` | Create ad-hoc error |

## Library Errors with thiserror

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApiError {
    #[error("network request failed")]
    Network(#[from] reqwest::Error),

    #[error("invalid response: status {status}")]
    InvalidResponse { status: u16 },

    #[error("resource not found: {0}")]
    NotFound(String),

    #[error("authentication failed")]
    Unauthorized,

    #[error("rate limited, retry after {retry_after_secs}s")]
    RateLimited { retry_after_secs: u64 },

    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

// Convenience alias
pub type Result<T> = std::result::Result<T, ApiError>;
```

### thiserror Attributes

| Attribute | Purpose |
|-----------|---------|
| `#[error("msg")]` | Display implementation |
| `#[error("msg {field}")]` | Interpolate fields |
| `#[error(transparent)]` | Delegate Display to inner |
| `#[from]` | Auto-implement `From<T>` |
| `#[source]` | Mark as error source (for chaining) |

## Error Wrapping Patterns

### Adding Context (Application)

```rust
// Good: context explains what was being done
let file = File::open(path)
    .context("opening database config")?;

// Bad: just restating the error
let file = File::open(path)
    .context("failed to open file")?;
```

### Mapping Errors (Library)

```rust
// Convert between error types
fn parse_port(s: &str) -> Result<u16, ConfigError> {
    s.parse::<u16>()
        .map_err(|_| ConfigError::InvalidPort(s.to_string()))
}
```

### Error Chain

```rust
// Inspect the chain
fn handle_error(err: &anyhow::Error) {
    eprintln!("Error: {err}");
    for cause in err.chain().skip(1) {
        eprintln!("  Caused by: {cause}");
    }
}
```

## Pattern Matching on Errors

```rust
// With thiserror enums
match result {
    Ok(data) => process(data),
    Err(ApiError::NotFound(id)) => eprintln!("not found: {id}"),
    Err(ApiError::RateLimited { retry_after_secs }) => {
        tokio::time::sleep(Duration::from_secs(retry_after_secs)).await;
        retry().await
    }
    Err(e) => return Err(e),
}

// Downcast anyhow errors
if let Some(api_err) = err.downcast_ref::<ApiError>() {
    match api_err {
        ApiError::Unauthorized => redirect_to_login(),
        _ => show_error(api_err),
    }
}
```

## Dioxus Error Handling

### Component Errors

```rust
#[component]
fn UserProfile(id: u64) -> Element {
    let user = use_resource(move || async move {
        get_user(id).await
    });

    match &*user.read() {
        Some(Ok(user)) => rsx! {
            h1 { "{user.name}" }
        },
        Some(Err(e)) => rsx! {
            div { class: "error", "Failed to load user: {e}" }
        },
        None => rsx! {
            div { class: "loading", "Loading..." }
        },
    }
}
```

### Server Function Errors

```rust
#[server]
async fn create_post(title: String, body: String) -> Result<Post, ServerFnError> {
    if title.is_empty() {
        return Err(ServerFnError::new("title cannot be empty"));
    }

    let post = db::insert_post(&title, &body)
        .await
        .map_err(|e| ServerFnError::new(e.to_string()))?;

    Ok(post)
}
```

## Recovery Patterns

### Default Values

```rust
// Provide defaults for non-critical failures
let config = load_config("app.toml").unwrap_or_default();
let port = env::var("PORT")
    .ok()
    .and_then(|s| s.parse().ok())
    .unwrap_or(8080);
```

### Retry with Backoff

```rust
async fn fetch_with_retry(url: &str, max_retries: u32) -> Result<Response> {
    let mut delay = Duration::from_millis(100);

    for attempt in 0..max_retries {
        match reqwest::get(url).await {
            Ok(resp) if resp.status().is_success() => return Ok(resp),
            Ok(resp) if resp.status() == 429 => {
                tokio::time::sleep(delay).await;
                delay *= 2;
            }
            Ok(resp) => bail!("unexpected status: {}", resp.status()),
            Err(e) if attempt < max_retries - 1 => {
                tokio::time::sleep(delay).await;
                delay *= 2;
            }
            Err(e) => return Err(e.into()),
        }
    }

    bail!("max retries exceeded for {url}")
}
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| `unwrap()` in production | Use `?`, `expect("reason")`, or handle |
| `expect("")` with empty message | Document the invariant: `expect("config loaded at startup")` |
| Logging AND returning error | Do one or the other, never both |
| `Box<dyn Error>` in library | Use `thiserror` for typed errors |
| Matching on error strings | Use typed errors with `thiserror` |
| `panic!` for expected failures | Return `Result` |
| Ignoring errors with `let _ =` | Handle or explicitly comment why it's safe |

## Sources

- [Rust By Example -- Error Handling](https://doc.rust-lang.org/stable/rust-by-example/error.html)
- [thiserror documentation](https://docs.rs/thiserror/)
- [anyhow documentation](https://docs.rs/anyhow/)
- [Rust API Guidelines -- Errors](https://rust-lang.github.io/api-guidelines/interoperability.html)
