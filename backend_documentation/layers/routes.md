# Routes Layer

**Location:** `src/routes/`

Routes are the entry points of the API. Each file in this module defines a function that returns an `axum::Router<AppState>` — a collection of URL patterns bound to handler functions. The routers are merged together in `main.rs`.

---

## Responsibilities

- Declare which HTTP method + path maps to which handler.
- Group related endpoints into one router function per domain.
- Keep routing separate from request-handling logic (no business logic here).

Routes do **not** contain middleware configuration; protection is applied per-handler via extractors (see [Authentication](../auth.md)).

---

## How Routers Are Assembled

In `src/main.rs`:

```rust
let app = Router::new()
    .merge(routes::user::users_router())
    .merge(routes::auth::auth_router())
    .merge(routes::project::projects_router())
    .with_state(state.clone());
```

Each domain module exposes a single `*_router()` function. Adding a new domain means creating a new `src/routes/<domain>.rs` file, implementing a `<domain>_router()` function, and merging it here.

---

## Endpoint Reference

### Auth — `src/routes/auth.rs`

| Method | Path             | Handler              | Auth required |
|--------|------------------|----------------------|---------------|
| POST   | `/auth/register` | `handlers::auth::register` | No  |
| POST   | `/auth/login`    | `handlers::auth::login`    | No  |

### Users — `src/routes/user.rs`

| Method | Path          | Handler                  | Auth required |
|--------|---------------|--------------------------|---------------|
| GET    | `/users`      | `handlers::user::get_users` | Yes (`AuthUser`) |
| GET    | `/user/{id}`  | `handlers::user::get_user`  | Yes (`AuthUser`) |

`{id}` is a UUID path parameter extracted automatically by Axum.

### Projects — `src/routes/project.rs`

| Method | Path               | Handler                        | Auth required |
|--------|--------------------|--------------------------------|---------------|
| POST   | `/projects/create` | `handlers::project::create_project` | Yes (`AuthUser`) |
| GET    | `/projects/get`    | `handlers::project::get_projects`   | Yes (`AuthUser`) |

The following routes are stubbed but currently commented out:

```
GET    /projects/get/{id}      — get single project
PUT    /projects/update/{id}   — update project
DELETE /projects/delete/{id}   — delete project
```

---

## Adding a New Route

1. Create (or open) `src/routes/<domain>.rs`.
2. Declare a `pub fn <domain>_router() -> Router<AppState>` function.
3. Add `.route("/path", method(handler::fn))` entries.
4. Merge the router in `src/main.rs`.
5. If the endpoint needs authentication, add `AuthUser` as a handler parameter (no route-level change needed).
