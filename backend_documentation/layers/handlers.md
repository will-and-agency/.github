# Handlers Layer

**Location:** `src/handlers/`

Handlers are async functions that Axum calls when a route is matched. They are the bridge between the HTTP layer and the rest of the application. Each handler:

1. Extracts data from the request (state, path params, JSON body, auth claims).
2. Runs a database query or calls a service function.
3. Returns a typed `Result<Json<T>, AppError>`.

---

## Axum Extractor Pattern

Axum uses function parameter types to extract data from the request. Order matters for non-`FromRequestParts` extractors (those that consume the body must come last).

```rust
pub async fn create_project(
    State(state): State<AppState>,       // shared application state
    AuthUser(claims): AuthUser,          // JWT claims (runs auth middleware)
    Valid(Json(payload)): Valid<Json<CreateProjectDto>>, // validated JSON body
) -> Result<Json<projects::Model>, AppError>
```

Common extractors used in this project:

| Extractor                   | Source                  | Purpose                                     |
|-----------------------------|-------------------------|---------------------------------------------|
| `State(state): State<AppState>` | `axum`              | Access DB and config                        |
| `AuthUser(claims): AuthUser`    | `middleware::auth`  | Validate JWT and inject claims              |
| `Json(body): Json<T>`           | `axum`              | Deserialise request body                    |
| `Valid(Json(dto)): Valid<Json<T>>` | `axum-valid`      | Deserialise + validate request body         |
| `Path(id): Path<Uuid>`          | `axum`              | Extract a typed path parameter              |

---

## Handler Catalogue

### `src/handlers/auth.rs`

#### `register`

Creates a new account. Returns a JWT on success.

```
POST /auth/register
Body: { email, password, first_name, last_name, role }
Response 200: { token: String }
Response 400: "Email already in use"
Response 500: database / hashing error
```

Flow:
1. Query `accounts` for an existing row with the same email.
2. If found → return `400 Bad Request`.
3. Hash the password with bcrypt.
4. Insert a new `accounts::ActiveModel` (UUID v4 primary key).
5. Generate and return a JWT.

#### `login`

Authenticates an existing account. Returns a JWT on success.

```
POST /auth/login
Body: { email, password }
Response 200: { token: String }
Response 400: "Invalid password"
Response 404: "Account not found"
Response 500: database / hashing error
```

Flow:
1. Query `accounts` by email.
2. If not found → `404 Not Found`.
3. Verify the supplied password against the stored bcrypt hash.
4. If invalid → `400 Bad Request`.
5. Generate and return a JWT.

---

### `src/handlers/user.rs`

All user routes require a valid JWT (`AuthUser` extractor).

#### `get_users`

```
GET /users
Auth: Bearer token required
Response 200: Array<accounts::Model>
```

Fetches all rows from the `accounts` table.

#### `get_user`

```
GET /user/{id}
Auth: Bearer token required
Response 200: accounts::Model
Response 404: "User not found"
```

Fetches a single account by UUID primary key.

---

### `src/handlers/project.rs`

All project routes require a valid JWT.

#### `get_projects`

```
GET /projects/get
Auth: Bearer token required
Response 200: Array<projects::Model>
```

Fetches all rows from the `projects` table.

#### `create_project`

```
POST /projects/create
Auth: Bearer token required
Body: CreateProjectDto (see DTOs)
Response 200: projects::Model
Response 422: validation error (field constraints / date logic)
Response 500: database error
```

Flow:
1. `AuthUser` validates the JWT and injects `claims` (contains the caller's UUID).
2. `Valid(Json(payload))` deserialises and validates the body using `CreateProjectDto`.
3. Builds a `projects::ActiveModel` — `created_by` is set from `claims.sub`.
4. Inserts and returns the created project.

---

## Conventions

- Handlers return `Result<Json<T>, AppError>`. The `?` operator on `sea_orm::DbErr` works because `AppError` implements `From<sea_orm::DbErr>`.
- Handlers do **not** contain reusable business logic — that belongs in `services/`.
- Inline request/response structs (like `RegisterRequest`) are acceptable in handlers for simple, one-off shapes. Complex or reusable input shapes go in `dto/`.
