# Rust Testing Patterns Reference

Unit tests, integration tests, property-based testing, mocking, and Dioxus component tests.

## Unit Tests

Place `#[cfg(test)] mod tests` at the bottom of each file:

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

pub fn divide(a: f64, b: f64) -> Result<f64, &'static str> {
    if b == 0.0 {
        return Err("division by zero");
    }
    Ok(a / b)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_positive_numbers() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn add_negative_numbers() {
        assert_eq!(add(-2, -3), -5);
    }

    #[test]
    fn divide_normal() {
        let result = divide(10.0, 3.0).unwrap();
        assert!((result - 3.333).abs() < 0.001);
    }

    #[test]
    fn divide_by_zero_returns_error() {
        assert!(divide(10.0, 0.0).is_err());
    }

    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn panics_on_invalid_index() {
        let v = vec![1, 2, 3];
        let _ = v[10];
    }
}
```

## Assertion Macros

| Macro | Purpose |
|-------|---------|
| `assert_eq!(a, b)` | Equality (shows both values on failure) |
| `assert_ne!(a, b)` | Inequality |
| `assert!(cond)` | Boolean condition |
| `assert!(result.is_ok())` | Result is Ok |
| `assert!(result.is_err())` | Result is Err |
| `assert_matches!(val, Pattern)` | Pattern matching (nightly or `matches!`) |
| `#[should_panic]` | Test expects a panic |
| `#[should_panic(expected = "msg")]` | Panic with specific message |

## Async Tests

Using `tokio::test`:

```rust
#[tokio::test]
async fn fetch_user_returns_data() {
    let client = TestClient::new().await;
    let user = client.get_user(1).await.unwrap();
    assert_eq!(user.name, "Alice");
}

#[tokio::test]
async fn concurrent_operations_succeed() {
    let (a, b) = tokio::join!(
        fetch_data("endpoint_a"),
        fetch_data("endpoint_b"),
    );
    assert!(a.is_ok());
    assert!(b.is_ok());
}
```

## Integration Tests

Place in `tests/` directory:

```rust
// tests/api_test.rs
use my_app::server::create_app;

#[tokio::test]
async fn health_check_returns_ok() {
    let app = create_app().await;
    let response = app.get("/health").await;
    assert_eq!(response.status(), 200);
}
```

## Test Organization Patterns

### Test Helpers

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Helper: build test data
    fn sample_user() -> User {
        User {
            id: 1,
            name: "Alice".into(),
            email: "alice@example.com".into(),
        }
    }

    // Helper: setup test state
    fn setup() -> (Database, Config) {
        let config = Config::test_defaults();
        let db = Database::in_memory(&config).unwrap();
        (db, config)
    }

    #[test]
    fn user_display_name() {
        let user = sample_user();
        assert_eq!(user.display_name(), "Alice");
    }
}
```

### Test Fixtures with tempfile

```rust
#[cfg(test)]
mod tests {
    use tempfile::TempDir;

    #[test]
    fn saves_config_to_file() {
        let dir = TempDir::new().unwrap();
        let path = dir.path().join("config.toml");

        let config = Config::default();
        config.save(&path).unwrap();

        let loaded = Config::load(&path).unwrap();
        assert_eq!(config, loaded);
    }
}
```

## Mocking

### Trait-Based Mocking (mockall)

```rust
use mockall::automock;

#[automock]
trait UserRepository {
    async fn find_by_id(&self, id: u64) -> Result<User>;
    async fn save(&self, user: &User) -> Result<()>;
}

#[tokio::test]
async fn service_returns_user() {
    let mut mock = MockUserRepository::new();
    mock.expect_find_by_id()
        .with(eq(1))
        .returning(|_| Ok(User { id: 1, name: "Alice".into() }));

    let service = UserService::new(mock);
    let user = service.get_user(1).await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

### Simple Test Doubles

```rust
// For simple cases, a manual test double is cleaner than mockall
struct FakeEmailSender {
    sent: RefCell<Vec<(String, String)>>,
}

impl EmailSender for FakeEmailSender {
    fn send(&self, to: &str, body: &str) -> Result<()> {
        self.sent.borrow_mut().push((to.into(), body.into()));
        Ok(())
    }
}
```

## Property-Based Testing (proptest)

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn parse_and_format_roundtrip(s in "[a-zA-Z0-9_]{1,50}") {
        let parsed = parse_identifier(&s).unwrap();
        let formatted = format_identifier(&parsed);
        assert_eq!(s, formatted);
    }

    #[test]
    fn add_is_commutative(a in -1000i32..1000, b in -1000i32..1000) {
        assert_eq!(add(a, b), add(b, a));
    }

    #[test]
    fn sort_preserves_length(mut v in prop::collection::vec(any::<i32>(), 0..100)) {
        let original_len = v.len();
        v.sort();
        assert_eq!(v.len(), original_len);
    }
}
```

## Snapshot Testing (insta)

```rust
use insta::assert_snapshot;

#[test]
fn renders_user_card_html() {
    let html = render_user_card(&User {
        name: "Alice".into(),
        email: "alice@example.com".into(),
    });
    assert_snapshot!(html);
}

#[test]
fn serializes_config() {
    let config = Config::default();
    insta::assert_json_snapshot!(config);
}
```

Manage snapshots with `cargo insta review`.

## Benchmarks (criterion)

```rust
// benches/benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use my_crate::process;

fn bench_process(c: &mut Criterion) {
    let data = vec![0u8; 1024];
    c.bench_function("process 1KB", |b| {
        b.iter(|| process(black_box(&data)))
    });
}

criterion_group!(benches, bench_process);
criterion_main!(benches);
```

## Doc Tests

```rust
/// Adds two numbers.
///
/// # Examples
///
/// ```
/// use my_crate::add;
/// assert_eq!(add(2, 3), 5);
/// ```
///
/// ```
/// use my_crate::add;
/// assert_eq!(add(-1, 1), 0);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Doc tests are compiled and run with `cargo test --doc`.

## Quick Reference

```bash
cargo test                              # all tests
cargo test test_name                    # specific test by name
cargo test -- --nocapture               # show println output
cargo test --lib                        # unit tests only
cargo test --test integration_test      # specific integration test
cargo test --doc                        # doc tests only
cargo test -- --test-threads=1          # run tests sequentially
cargo bench                             # benchmarks
cargo tarpaulin --out html              # coverage report
cargo insta review                      # review snapshots
```

## Test Dependencies

Add test-only dependencies in `Cargo.toml`:

```toml
[dev-dependencies]
tokio = { version = "1", features = ["test-util", "macros"] }
mockall = "0.13"
proptest = "1"
tempfile = "3"
insta = { version = "1", features = ["json"] }
criterion = "0.5"
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Testing implementation details | Test public API behavior |
| Tests depend on execution order | Each test sets up its own state |
| Flaky tests (timing-dependent) | Use deterministic approaches, mock time |
| Giant test functions | Split into focused, named test functions |
| `unwrap()` in tests without context | Use `unwrap()` is OK in tests, but `.expect("reason")` is better |
| No error case tests | Always test error paths, not just happy paths |

## Sources

- [Rust By Example -- Testing](https://doc.rust-lang.org/stable/rust-by-example/testing.html)
- [The Rust Programming Language -- Testing](https://doc.rust-lang.org/stable/book/ch11-00-testing.html)
- [proptest documentation](https://docs.rs/proptest/)
- [mockall documentation](https://docs.rs/mockall/)
- [insta documentation](https://docs.rs/insta/)
