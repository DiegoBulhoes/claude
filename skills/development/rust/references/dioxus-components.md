# Dioxus Component Patterns Reference

Component architecture, props, children, memoization, and composition patterns.

## Basic Component

```rust
use dioxus::prelude::*;

#[component]
fn Greeting(name: String) -> Element {
    rsx! {
        h1 { "Hello, {name}!" }
    }
}

// Usage:
rsx! { Greeting { name: "Alice" } }
```

The `#[component]` macro:
- Converts function parameters into a `Props` struct
- Auto-derives `Clone + PartialEq` on props
- Enables memoization (re-render only when props change)

## Props Patterns

### Optional Props

```rust
#[component]
fn Button(
    label: String,
    #[props(default = false)] disabled: bool,
    #[props(default = "primary".to_string())] variant: String,
    #[props(default)] onclick: EventHandler<MouseEvent>,
) -> Element {
    rsx! {
        button {
            class: "btn btn-{variant}",
            disabled: disabled,
            onclick: move |e| onclick.call(e),
            "{label}"
        }
    }
}

// All valid:
rsx! {
    Button { label: "Click me" }
    Button { label: "Submit", variant: "success" }
    Button { label: "Delete", variant: "danger", onclick: move |_| delete() }
}
```

### Children Props

```rust
#[component]
fn Card(title: String, children: Element) -> Element {
    rsx! {
        div { class: "card",
            h2 { class: "card-title", "{title}" }
            div { class: "card-body", {children} }
        }
    }
}

// Usage:
rsx! {
    Card { title: "User Info",
        p { "Name: Alice" }
        p { "Email: alice@example.com" }
    }
}
```

### Manual Props (for Libraries)

```rust
#[derive(Props, PartialEq, Clone)]
pub struct DataTableProps {
    pub headers: Vec<String>,
    pub rows: Vec<Vec<String>>,
    #[props(default = true)]
    pub striped: bool,
    #[props(default)]
    pub on_row_click: EventHandler<usize>,
}

fn DataTable(props: DataTableProps) -> Element {
    rsx! {
        table { class: if props.striped { "table-striped" },
            thead {
                tr {
                    for header in &props.headers {
                        th { "{header}" }
                    }
                }
            }
            tbody {
                for (i, row) in props.rows.iter().enumerate() {
                    tr {
                        onclick: move |_| props.on_row_click.call(i),
                        for cell in row {
                            td { "{cell}" }
                        }
                    }
                }
            }
        }
    }
}
```

## Event Handling

### Common Events

```rust
#[component]
fn Form() -> Element {
    let mut name = use_signal(|| String::new());
    let mut submitted = use_signal(|| false);

    rsx! {
        form {
            // Prevent default form submission
            onsubmit: move |e| {
                e.prevent_default();
                submitted.set(true);
            },

            input {
                r#type: "text",
                value: "{name}",
                oninput: move |e| name.set(e.value()),
                placeholder: "Enter name",
            }

            button { r#type: "submit", "Submit" }
        }

        if *submitted.read() {
            p { "Submitted: {name}" }
        }
    }
}
```

### Async Event Handlers

```rust
#[component]
fn SearchBox() -> Element {
    let mut query = use_signal(|| String::new());
    let mut results = use_signal(|| Vec::<String>::new());

    rsx! {
        input {
            value: "{query}",
            oninput: move |e| {
                query.set(e.value());
            },
        }
        button {
            onclick: move |_| async move {
                let data = search_api(&query.read()).await.unwrap_or_default();
                results.set(data);
            },
            "Search"
        }
        ul {
            for result in results.read().iter() {
                li { "{result}" }
            }
        }
    }
}
```

## Conditional Rendering

```rust
#[component]
fn Dashboard(user: Option<User>) -> Element {
    rsx! {
        // if/else
        if let Some(user) = &user {
            h1 { "Welcome, {user.name}" }
            if user.is_admin {
                AdminPanel {}
            }
        } else {
            LoginForm {}
        }

        // Optional elements
        {user.as_ref().map(|u| rsx! {
            p { "Last login: {u.last_login}" }
        })}
    }
}
```

## List Rendering

```rust
#[component]
fn UserList(users: Vec<User>) -> Element {
    rsx! {
        ul {
            // MUST provide key for efficient diffing
            for user in &users {
                li { key: "{user.id}",
                    UserCard { user: user.clone() }
                }
            }
        }

        // Empty state
        if users.is_empty() {
            p { class: "empty", "No users found" }
        }
    }
}
```

Keys MUST be:
- Unique among siblings
- Stable across re-renders (NOT index-based for dynamic lists)
- Derived from data identity (ID, unique field)

## Component Composition

### Layout Components

```rust
#[component]
fn PageLayout(title: String, children: Element) -> Element {
    rsx! {
        div { class: "page",
            Header { title: title.clone() }
            main { class: "content", {children} }
            Footer {}
        }
    }
}

#[component]
fn TwoColumnLayout(sidebar: Element, children: Element) -> Element {
    rsx! {
        div { class: "two-column",
            aside { class: "sidebar", {sidebar} }
            main { class: "main", {children} }
        }
    }
}
```

### Render Props Pattern

```rust
#[component]
fn DataLoader<T: Clone + PartialEq + 'static>(
    fetch: AsyncFn<T>,
    render: fn(T) -> Element,
) -> Element {
    let data = use_resource(move || fetch());

    match &*data.read() {
        Some(Ok(data)) => render(data.clone()),
        Some(Err(e)) => rsx! { div { class: "error", "Error: {e}" } },
        None => rsx! { div { class: "loading", "Loading..." } },
    }
}
```

## Custom Hooks

```rust
/// Hook for managing form state with validation
fn use_form_field(initial: &str) -> (Signal<String>, Signal<Option<String>>, bool) {
    let value = use_signal(|| initial.to_string());
    let error = use_signal(|| None::<String>);
    let is_valid = use_memo(move || error.read().is_none() && !value.read().is_empty());

    (value, error, *is_valid.read())
}

/// Hook for debounced search
fn use_debounced_search(delay_ms: u64) -> (Signal<String>, Signal<Vec<SearchResult>>) {
    let query = use_signal(|| String::new());
    let results = use_signal(|| Vec::new());

    use_effect(move || {
        let q = query.read().clone();
        spawn(async move {
            tokio::time::sleep(Duration::from_millis(delay_ms)).await;
            if *query.read() == q {
                if let Ok(data) = search(&q).await {
                    results.set(data);
                }
            }
        });
    });

    (query, results)
}
```

## CSS and Styling

### Class-Based Styling

```rust
#[component]
fn Alert(variant: String, message: String) -> Element {
    let class = format!("alert alert-{variant}");
    rsx! {
        div { class: class, "{message}" }
    }
}
```

### Dynamic Styles

```rust
#[component]
fn ProgressBar(percent: f64) -> Element {
    rsx! {
        div { class: "progress-bar",
            div {
                class: "progress-fill",
                width: "{percent}%",
                background_color: if percent > 80.0 { "green" } else { "blue" },
            }
        }
    }
}
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Missing `key` in lists | Always provide stable, unique keys |
| Calling hooks conditionally | Move hooks to top of component |
| Giant monolithic components | Split into focused components |
| Props drilling through many layers | Use `use_context` for deeply shared state |
| Cloning large data in props | Use `Signal<T>` or `Arc<T>` for shared data |
| Business logic in components | Extract to functions or hooks |

## Sources

- [Dioxus Component Docs](https://dioxuslabs.com/learn/0.6/)
- [Dioxus API Reference](https://docs.rs/dioxus/latest/dioxus/)
