# Dioxus State Management Reference

Signals, context, derived state, async data, and server functions.

## Signals (Local State)

### use_signal -- Primary Reactive State

```rust
#[component]
fn Counter() -> Element {
    let mut count = use_signal(|| 0);

    rsx! {
        p { "Count: {count}" }
        button { onclick: move |_| count += 1, "+" }
        button { onclick: move |_| count -= 1, "-" }
        button { onclick: move |_| count.set(0), "Reset" }
    }
}
```

Signal operations:

| Operation | Syntax | Notes |
|-----------|--------|-------|
| Read | `{count}` in RSX, `*count.read()` in code | Auto-subscribes component |
| Write | `count.set(value)` | Replaces value |
| Modify | `count += 1`, `count -= 1` | Arithmetic operators |
| Mutate | `count.write().field = val` | For complex types |
| Peek | `count.peek()` | Read WITHOUT subscribing |

### use_hook -- Non-Reactive Stored Value

```rust
#[component]
fn Logger() -> Element {
    // Stored but does NOT trigger re-render when changed
    let log_count = use_hook(|| RefCell::new(0));

    rsx! {
        button {
            onclick: move |_| {
                *log_count.borrow_mut() += 1;
                println!("Clicked {} times", log_count.borrow());
            },
            "Log"
        }
    }
}
```

## Derived State

### use_memo -- Computed Values

```rust
#[component]
fn FilteredList(items: Vec<Item>, filter: String) -> Element {
    // Recomputes only when items or filter change
    let filtered = use_memo(move || {
        items.iter()
            .filter(|i| i.name.contains(&*filter.read()))
            .cloned()
            .collect::<Vec<_>>()
    });

    rsx! {
        ul {
            for item in filtered.read().iter() {
                li { key: "{item.id}", "{item.name}" }
            }
        }
    }
}
```

## Side Effects

### use_effect

```rust
#[component]
fn DocumentTitle(title: String) -> Element {
    // Runs after every render where `title` changed
    use_effect(move || {
        web_sys::window()
            .and_then(|w| w.document())
            .map(|d| d.set_title(&title.read()));
    });

    rsx! { div {} }
}
```

## Global State

### GlobalSignal

```rust
// Define at module level -- accessible from any component
static THEME: GlobalSignal<String> = Signal::global(|| "light".to_string());
static AUTH_TOKEN: GlobalSignal<Option<String>> = Signal::global(|| None);

#[component]
fn ThemeToggle() -> Element {
    rsx! {
        button {
            onclick: move |_| {
                let current = THEME.read().clone();
                *THEME.write() = if current == "light" { "dark".into() } else { "light".into() };
            },
            "Toggle Theme (current: {THEME})"
        }
    }
}

#[component]
fn AuthStatus() -> Element {
    rsx! {
        if AUTH_TOKEN.read().is_some() {
            p { "Logged in" }
        } else {
            p { "Not logged in" }
        }
    }
}
```

Use GlobalSignal for:
- Theme preferences
- Authentication state
- Feature flags
- App-wide settings

## Context API (Shared State)

### Providing Context

```rust
#[derive(Clone, Copy)]
struct AppState {
    user: Signal<Option<User>>,
    notifications: Signal<Vec<Notification>>,
    sidebar_open: Signal<bool>,
}

#[component]
fn App() -> Element {
    // Provide state to all descendants
    use_context_provider(|| AppState {
        user: Signal::new(None),
        notifications: Signal::new(Vec::new()),
        sidebar_open: Signal::new(true),
    });

    rsx! { Router::<Route> {} }
}
```

### Consuming Context

```rust
#[component]
fn UserMenu() -> Element {
    let state = use_context::<AppState>();

    match &*state.user.read() {
        Some(user) => rsx! {
            div { class: "user-menu",
                span { "{user.name}" }
                button {
                    onclick: move |_| state.user.set(None),
                    "Logout"
                }
            }
        },
        None => rsx! {
            Link { to: Route::Login {}, "Login" }
        },
    }
}

#[component]
fn NotificationBadge() -> Element {
    let state = use_context::<AppState>();
    let count = state.notifications.read().len();

    rsx! {
        if count > 0 {
            span { class: "badge", "{count}" }
        }
    }
}
```

### Context vs GlobalSignal

| Feature | Context | GlobalSignal |
|---------|---------|-------------|
| Scope | Subtree (descendants of provider) | Global (entire app) |
| Multiple instances | Yes (different providers at different levels) | No (one per static) |
| Testing | Easy to mock (provide test context) | Harder (global state) |
| Use case | Component tree state | App-wide singletons |

## Async Data Fetching

### use_resource

```rust
#[component]
fn UserProfile(id: u64) -> Element {
    // Automatically re-fetches when `id` changes
    let user = use_resource(move || async move {
        reqwest::get(format!("https://api.example.com/users/{id}"))
            .await?
            .json::<User>()
            .await
    });

    match &*user.read() {
        Some(Ok(user)) => rsx! {
            div {
                h1 { "{user.name}" }
                p { "{user.email}" }
                button {
                    onclick: move |_| user.restart(),
                    "Refresh"
                }
            }
        },
        Some(Err(e)) => rsx! { p { class: "error", "Error: {e}" } },
        None => rsx! { p { "Loading..." } },
    }
}
```

### Manual Async (spawn)

```rust
#[component]
fn SubmitForm() -> Element {
    let mut status = use_signal(|| "idle");
    let mut name = use_signal(|| String::new());

    rsx! {
        input {
            value: "{name}",
            oninput: move |e| name.set(e.value()),
        }
        button {
            disabled: *status.read() == "loading",
            onclick: move |_| async move {
                status.set("loading");
                match submit(&name.read()).await {
                    Ok(_) => status.set("success"),
                    Err(_) => status.set("error"),
                }
            },
            match *status.read() {
                "loading" => "Submitting...",
                "success" => "Done!",
                "error" => "Failed - Retry",
                _ => "Submit",
            }
        }
    }
}
```

## Server Functions

### Basic Server Function

```rust
#[server]
async fn get_todos() -> Result<Vec<Todo>, ServerFnError> {
    let pool = get_db_pool().await?;
    let todos = sqlx::query_as!(Todo, "SELECT * FROM todos ORDER BY created_at DESC")
        .fetch_all(&pool)
        .await
        .map_err(|e| ServerFnError::new(e.to_string()))?;
    Ok(todos)
}

#[server]
async fn create_todo(title: String) -> Result<Todo, ServerFnError> {
    if title.trim().is_empty() {
        return Err(ServerFnError::new("title cannot be empty"));
    }

    let pool = get_db_pool().await?;
    let todo = sqlx::query_as!(
        Todo,
        "INSERT INTO todos (title) VALUES ($1) RETURNING *",
        title.trim()
    )
    .fetch_one(&pool)
    .await
    .map_err(|e| ServerFnError::new(e.to_string()))?;

    Ok(todo)
}
```

### Using Server Functions in Components

```rust
#[component]
fn TodoApp() -> Element {
    let todos = use_resource(|| get_todos());
    let mut new_title = use_signal(|| String::new());

    rsx! {
        form {
            onsubmit: move |_| async move {
                if let Ok(todo) = create_todo(new_title.read().clone()).await {
                    new_title.set(String::new());
                    todos.restart(); // Refresh the list
                }
            },
            input {
                value: "{new_title}",
                oninput: move |e| new_title.set(e.value()),
                placeholder: "New todo...",
            }
            button { "Add" }
        }

        match &*todos.read() {
            Some(Ok(items)) => rsx! {
                ul {
                    for todo in items {
                        li { key: "{todo.id}", "{todo.title}" }
                    }
                }
            },
            Some(Err(e)) => rsx! { p { "Error: {e}" } },
            None => rsx! { p { "Loading..." } },
        }
    }
}
```

## State Management Patterns

### Form State

```rust
#[derive(Clone, Default, PartialEq)]
struct FormData {
    name: String,
    email: String,
    message: String,
}

#[component]
fn ContactForm() -> Element {
    let mut form = use_signal(FormData::default);
    let mut errors = use_signal(|| HashMap::<String, String>::new());

    let validate = move || {
        let mut errs = HashMap::new();
        let data = form.read();
        if data.name.is_empty() { errs.insert("name".into(), "required".into()); }
        if !data.email.contains('@') { errs.insert("email".into(), "invalid email".into()); }
        errors.set(errs.clone());
        errs.is_empty()
    };

    rsx! {
        form {
            onsubmit: move |e| async move {
                e.prevent_default();
                if validate() {
                    submit_form(&form.read()).await.ok();
                }
            },
            div {
                label { "Name" }
                input {
                    value: "{form.read().name}",
                    oninput: move |e| form.write().name = e.value(),
                }
                if let Some(err) = errors.read().get("name") {
                    span { class: "error", "{err}" }
                }
            }
            // ... more fields
            button { "Submit" }
        }
    }
}
```

### Loading/Error/Success State Machine

```rust
#[derive(Clone, PartialEq)]
enum AsyncState<T: Clone + PartialEq> {
    Idle,
    Loading,
    Success(T),
    Error(String),
}

fn use_async_action<T, F, Fut>(action: F) -> (Signal<AsyncState<T>>, impl Fn())
where
    T: Clone + PartialEq + 'static,
    F: Fn() -> Fut + 'static,
    Fut: Future<Output = Result<T, anyhow::Error>> + 'static,
{
    let state = use_signal(|| AsyncState::<T>::Idle);

    let trigger = move || {
        state.set(AsyncState::Loading);
        spawn(async move {
            match action().await {
                Ok(data) => state.set(AsyncState::Success(data)),
                Err(e) => state.set(AsyncState::Error(e.to_string())),
            }
        });
    };

    (state, trigger)
}
```

## Decision Table: Which State Tool?

| Need | Use | Why |
|------|-----|-----|
| Component-local toggle/counter | `use_signal` | Simplest reactive state |
| Value that doesn't trigger re-render | `use_hook` | Stored but not reactive |
| Computed/derived value | `use_memo` | Auto-recomputes, cached |
| Side effect on state change | `use_effect` | Runs after render |
| State shared in subtree | Context + `use_context` | Scoped, testable |
| App-wide singleton | `GlobalSignal` | No provider needed |
| Async data from API | `use_resource` | Suspense-integrated |
| Server-side data | `#[server]` + `use_resource` | RPC, auto-serialized |

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Signal in loop | Move signal above the loop, track collection |
| Conditional hook call | Always call hooks unconditionally at top |
| Reading signal in spawn without clone | Use `move` closure or `.read().clone()` |
| GlobalSignal for component state | Use `use_signal` -- global is for app-wide |
| Prop drilling through 5+ levels | Use Context API |
| Mutating signal during render | Mutate only in event handlers or effects |
| Not restarting resource after mutation | Call `resource.restart()` after server mutation |

## Sources

- [Dioxus State Management](https://dioxuslabs.com/learn/0.6/)
- [Dioxus Signals API](https://docs.rs/dioxus-signals/latest/dioxus_signals/)
- [Dioxus Hooks API](https://docs.rs/dioxus-hooks/latest/dioxus_hooks/)
- [Dioxus Fullstack](https://docs.rs/dioxus-fullstack/latest/dioxus_fullstack/)
