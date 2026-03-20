# Architecture Overview

## Layer Model

The backend is organised into four ordered layers. A request always flows top-to-bottom; each layer has one clear responsibility.

```
HTTP Request
     |
     v
+----------+
|  Routes  |   src/routes/      — declares which handler owns each URL
+----------+
     |
     v
+----------+
| Handlers |   src/handlers/    — parses the request, calls services/SeaORM, returns a response
+----------+
     |
     v
+----------+
| Services |   src/services/    — so far mainly auth functions (auth crypto, token generation, etc.)
+----------+
     |
     v
+----------+
| Entities |   src/entities/    — SeaORM models that map 1-to-1 to database tables
+----------+
     |
     v
  PostgreSQL
```

Middleware (authentication) runs **before** the handler via Axum's `FromRequestParts` extractor, not as a separate tower layer, so it is transparent to the route definition.

---

## Module Map

```
src/
├── main.rs              Entry point — wires config, DB, state, and router
├── config.rs            Reads env vars at startup; panics on missing required vars
├── db.rs                Establishes the SeaORM database connection
├── state.rs             AppState struct (DB connection + Config) shared across handlers
│
├── routes/
│   ├── mod.rs
│   ├── auth.rs          POST /auth/register, POST /auth/login
│   ├── user.rs          GET  /users, GET /user/{id}
│   └── project.rs       POST /projects/create, GET /projects/get
│
├── handlers/
│   ├── mod.rs
│   ├── auth.rs          register / login handlers
│   ├── user.rs          get_users / get_user handlers
│   └── project.rs       create_project / get_projects handlers
│
├── services/
│   ├── mod.rs
│   ├── auth.rs          hash_password, verify_password, generate_token, verify_token, Claims
│   ├── user.rs          (placeholder)
│   └── project.rs       (placeholder)
│
├── entities/
│   ├── mod.rs
│   ├── prelude.rs
│   ├── accounts.rs
│   ├── projects.rs
│   ├── tasks.rs
│   ├── questions.rs
│   ├── answers.rs
│   ├── attachments.rs
│   ├── moderators.rs
│   └── participants.rs
│
├── dto/
│   ├── mod.rs
│   └── projects.rs      CreateProjectDto (with validation)
│
├── middleware/
│   ├── mod.rs
│   └── auth.rs          AuthUser extractor — validates JWT on protected routes
│
└── errors.rs            AppError enum + IntoResponse impl
```

---

## Request Lifecycle

1. **Startup** — `main.rs` loads `Config`, connects to the database, builds `AppState`, and mounts routers.
2. **Routing** — Axum matches the incoming URL + method and selects a handler function.
3. **Middleware (auth)** — If the handler declares an `AuthUser` parameter, Axum automatically runs the `FromRequestParts` implementation, which reads the `Authorization: Bearer <token>` header and validates the JWT. A missing or invalid token short-circuits with a `401 Unauthorized` response before the handler body executes.
4. **Handler** — Deserialises the JSON body (optionally via a DTO with validation), interacts with SeaORM or calls a service function, and returns a `Result<Json<T>, AppError>`.
5. **Error conversion** — If the handler returns `Err(AppError)`, Axum calls `IntoResponse` on it, which maps it to the appropriate HTTP status code.
6. **Response** — Axum serialises the `Json<T>` value and writes the HTTP response.

---

## Shared State (`AppState`)

`AppState` is defined in `src/state.rs` and holds everything handlers need:

```rust
pub struct AppState {
    pub db: DatabaseConnection,  // SeaORM connection pool
    pub config: Config,          // Parsed environment config
}
```

It is cloned cheaply (both inner types are `Clone`) and injected into every handler via Axum's `State` extractor.
