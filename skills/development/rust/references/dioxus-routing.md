# Dioxus Routing Patterns Reference

Type-safe routing, nested routes, layouts, guards, and navigation.

## Basic Routing Setup

```rust
use dioxus::prelude::*;

#[derive(Routable, Clone, PartialEq)]
enum Route {
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
fn Home() -> Element {
    rsx! { h1 { "Home" } }
}

#[component]
fn About() -> Element {
    rsx! { h1 { "About" } }
}

#[component]
fn NotFound(segments: Vec<String>) -> Element {
    rsx! { h1 { "404 - Page Not Found" } }
}
```

## Dynamic Route Parameters

```rust
#[derive(Routable, Clone, PartialEq)]
enum Route {
    #[route("/")]
    Home {},

    // Single parameter
    #[route("/users/:id")]
    UserDetail { id: u64 },

    // Multiple parameters
    #[route("/posts/:category/:slug")]
    PostDetail { category: String, slug: String },

    // Query parameters
    #[route("/search?:query&:page")]
    Search { query: String, page: u32 },

    // Catch-all
    #[route("/:..segments")]
    NotFound { segments: Vec<String> },
}

#[component]
fn UserDetail(id: u64) -> Element {
    let user = use_resource(move || get_user(id));

    match &*user.read() {
        Some(Ok(user)) => rsx! { h1 { "{user.name}" } },
        Some(Err(e)) => rsx! { p { "Error: {e}" } },
        None => rsx! { p { "Loading..." } },
    }
}

#[component]
fn Search(query: String, page: u32) -> Element {
    rsx! {
        h1 { "Search: {query}" }
        p { "Page: {page}" }
    }
}
```

## Layouts and Nesting

```rust
#[derive(Routable, Clone, PartialEq)]
enum Route {
    // All routes under #[layout] share the NavBar layout
    #[layout(NavBar)]
        #[route("/")]
        Home {},
        #[route("/about")]
        About {},

        // Nested layout: Dashboard wraps admin routes
        #[layout(DashboardLayout)]
            #[route("/dashboard")]
            Dashboard {},
            #[route("/dashboard/settings")]
            Settings {},
        #[end_layout]
    #[end_layout]

    // Outside any layout
    #[route("/login")]
    Login {},

    #[route("/:..segments")]
    NotFound { segments: Vec<String> },
}

#[component]
fn NavBar() -> Element {
    rsx! {
        nav { class: "navbar",
            Link { to: Route::Home {}, "Home" }
            Link { to: Route::About {}, "About" }
            Link { to: Route::Dashboard {}, "Dashboard" }
        }
        // Child routes render here
        Outlet::<Route> {}
    }
}

#[component]
fn DashboardLayout() -> Element {
    rsx! {
        div { class: "dashboard",
            aside { class: "sidebar",
                Link { to: Route::Dashboard {}, "Overview" }
                Link { to: Route::Settings {}, "Settings" }
            }
            main {
                Outlet::<Route> {}
            }
        }
    }
}
```

## Navigation

### Link Component

```rust
#[component]
fn Navigation() -> Element {
    rsx! {
        nav {
            // Type-safe links (compile-time validated)
            Link { to: Route::Home {}, "Home" }
            Link { to: Route::UserDetail { id: 42 }, "User 42" }
            Link { to: Route::Search { query: "rust".into(), page: 1 },
                "Search Rust"
            }

            // Styled active link
            Link {
                to: Route::About {},
                class: "nav-link",
                active_class: "active",
                "About"
            }

            // External link
            a { href: "https://dioxuslabs.com", target: "_blank", "Dioxus" }
        }
    }
}
```

### Programmatic Navigation

```rust
#[component]
fn LoginForm() -> Element {
    let navigator = use_navigator();
    let mut username = use_signal(|| String::new());

    rsx! {
        form {
            onsubmit: move |_| async move {
                let success = login(&username.read()).await.unwrap_or(false);
                if success {
                    navigator.push(Route::Dashboard {});
                }
            },
            input {
                value: "{username}",
                oninput: move |e| username.set(e.value()),
            }
            button { "Login" }
        }
    }
}
```

### Navigation Methods

| Method | Purpose |
|--------|---------|
| `navigator.push(route)` | Navigate forward (adds to history) |
| `navigator.replace(route)` | Replace current entry (no back) |
| `navigator.go_back()` | Go to previous page |
| `navigator.go_forward()` | Go to next page |

## Route Guards (Authentication)

```rust
#[derive(Routable, Clone, PartialEq)]
enum Route {
    #[route("/")]
    Home {},
    #[route("/login")]
    Login {},
    #[layout(AuthGuard)]
        #[route("/dashboard")]
        Dashboard {},
        #[route("/profile")]
        Profile {},
    #[end_layout]
}

#[component]
fn AuthGuard() -> Element {
    let auth = use_context::<AuthState>();

    if auth.is_authenticated() {
        rsx! { Outlet::<Route> {} }
    } else {
        // Redirect to login
        let navigator = use_navigator();
        navigator.replace(Route::Login {});
        rsx! { p { "Redirecting..." } }
    }
}
```

## Active Route Detection

```rust
#[component]
fn SideNav() -> Element {
    let route = use_route::<Route>();

    let home_class = if matches!(route, Route::Home {}) { "active" } else { "" };
    let about_class = if matches!(route, Route::About {}) { "active" } else { "" };

    rsx! {
        nav {
            Link { to: Route::Home {}, class: "{home_class}", "Home" }
            Link { to: Route::About {}, class: "{about_class}", "About" }
        }
    }
}
```

## Error Boundaries for Routes

```rust
#[component]
fn UserDetail(id: u64) -> Element {
    let user = use_resource(move || async move {
        get_user(id).await
    });

    match &*user.read() {
        Some(Ok(user)) => rsx! {
            div { class: "user-detail",
                h1 { "{user.name}" }
                p { "{user.email}" }
            }
        },
        Some(Err(_)) => rsx! {
            div { class: "error",
                h1 { "User Not Found" }
                Link { to: Route::Home {}, "Go Home" }
            }
        },
        None => rsx! {
            div { class: "loading", "Loading user..." }
        },
    }
}
```

## Common Patterns

### Breadcrumbs

```rust
#[component]
fn Breadcrumbs() -> Element {
    let route = use_route::<Route>();

    let crumbs: Vec<(&str, Route)> = match &route {
        Route::Home {} => vec![("Home", Route::Home {})],
        Route::UserDetail { id } => vec![
            ("Home", Route::Home {}),
            ("Users", Route::Home {}),
            (&format!("User {id}"), Route::UserDetail { id: *id }),
        ],
        _ => vec![("Home", Route::Home {})],
    };

    rsx! {
        nav { class: "breadcrumbs",
            for (i, (label, route)) in crumbs.iter().enumerate() {
                if i > 0 { span { " / " } }
                Link { to: route.clone(), "{label}" }
            }
        }
    }
}
```

### Tab Navigation

```rust
#[derive(Routable, Clone, PartialEq)]
enum Route {
    #[layout(SettingsLayout)]
        #[route("/settings/general")]
        SettingsGeneral {},
        #[route("/settings/security")]
        SettingsSecurity {},
        #[route("/settings/notifications")]
        SettingsNotifications {},
    #[end_layout]
}

#[component]
fn SettingsLayout() -> Element {
    rsx! {
        div { class: "settings",
            div { class: "tabs",
                Link { to: Route::SettingsGeneral {}, active_class: "active", "General" }
                Link { to: Route::SettingsSecurity {}, active_class: "active", "Security" }
                Link { to: Route::SettingsNotifications {}, active_class: "active", "Notifications" }
            }
            div { class: "tab-content",
                Outlet::<Route> {}
            }
        }
    }
}
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| String-based routes | Use typed `Route` enum |
| Missing catch-all route | Add `#[route("/:..segments")]` for 404 |
| Auth check in every component | Use layout-based route guards |
| Deeply nested route enums | Keep route hierarchy flat, use layouts |
| Navigation without `use_navigator` | Use the hook for programmatic nav |
| Missing `Outlet` in layouts | Layout MUST include `Outlet::<Route> {}` |

## Sources

- [Dioxus Router Documentation](https://dioxuslabs.com/learn/0.6/)
- [Dioxus Router API](https://docs.rs/dioxus-router/latest/dioxus_router/)
