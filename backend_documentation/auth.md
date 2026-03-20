# Authentication

The backend uses **stateless JWT authentication**. There are no sessions or refresh tokens — every request to a protected endpoint must carry a valid `Authorization: Bearer <token>` header.

---

## Overview

```
Client                          Server
  |                               |
  |  POST /auth/login             |
  |  { email, password }          |
  |------------------------------>|
  |                               |  1. Verify password (bcrypt)
  |                               |  2. Generate JWT (HS256)
  |  { token: "eyJ..." }          |
  |<------------------------------|
  |                               |
  |  GET /users                   |
  |  Authorization: Bearer eyJ... |
  |------------------------------>|
  |                               |  3. AuthUser extractor validates JWT
  |                               |  4. Claims injected into handler
  |  [...users]                   |
  |<------------------------------|
```

---

## JWT Structure

Tokens are HS256-signed JWTs. The payload (`Claims`) contains:

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "role": "admin",
  "exp": 1710000000
}
```

| Field | Type   | Description                               |
|-------|--------|-------------------------------------------|
| `sub` | UUID   | The account ID (subject)                  |
| `role`| String | The account's role                        |
| `exp` | usize  | Unix timestamp — token expiry             |

---

## Service Functions (`src/services/auth.rs`)

### Password Hashing

```rust
// Hash at registration
let hash = hash_password(&body.password)?;

// Verify at login
let valid = verify_password(&body.password, &account.password_hash)?;
```

Uses the `bcrypt` crate at `DEFAULT_COST` (12 rounds). Hashes are stored in `accounts.password_hash`.

### Token Generation

```rust
let token = generate_token(
    account.id,          // Uuid — becomes claims.sub
    &account.role,       // String — becomes claims.role
    &config.jwt_secret,  // from JWT_SECRET env var
    config.jwt_expiry_hours, // from JWT_EXPIRY_HOURS env var (default: 24)
)?;
```

### Token Verification

```rust
let claims = verify_token(bearer.token(), &secret)?;
```

Verifies the signature and that the token has not expired. Returns `Err` on failure.

---

## Auth Middleware (`src/middleware/auth.rs`)

The `AuthUser` struct is an **Axum extractor** that implements `FromRequestParts`. It is not a traditional tower middleware layer — instead it is declared as a handler parameter and Axum runs it automatically before the handler body.

```rust
pub struct AuthUser(pub Claims);

impl<S> FromRequestParts<S> for AuthUser { ... }
```

### What it does

1. Reads the `Authorization: Bearer <token>` header using `TypedHeader<Authorization<Bearer>>`.
2. Returns `401 Unauthorized` if the header is missing or malformed.
3. Calls `verify_token` with the `JWT_SECRET` environment variable.
4. Returns `401 Unauthorized` if the token is invalid or expired.
5. Wraps the decoded `Claims` in `AuthUser` and passes it to the handler.

### Protecting a Route

Simply add `AuthUser(claims): AuthUser` as a parameter to any handler:

```rust
pub async fn my_handler(
    State(state): State<AppState>,
    AuthUser(claims): AuthUser,  // this line enforces authentication
) -> Result<Json<SomeResponse>, AppError> {
    // claims.sub = authenticated user's UUID
    // claims.role = their role
}
```

No changes to the router or any middleware stack are needed.

### Accessing the Caller's Identity

Inside a protected handler, `claims.sub` contains the authenticated account's UUID. This is used, for example, to set `created_by` when creating resources:

```rust
created_by: Set(claims.sub),
```

---

## Roles

Defined as constants in `src/services/auth.rs`:

```rust
Roles::ADMIN      // "admin"
Roles::USER       // "user"
Roles::MODERATOR  // "moderator"
```

Role checking within a handler:

```rust
if !claims.has_role(Roles::ADMIN) {
    return Err(AppError::BadRequest("Forbidden".to_string()));
}
```

Role-based access control (RBAC) is not yet enforced at the route level — it must be applied manually inside handlers that need it.

---

## Environment Variables

| Variable           | Required | Default | Description                     |
|--------------------|----------|---------|---------------------------------|
| `JWT_SECRET`       | Yes      | —       | HMAC signing secret             |
| `JWT_EXPIRY_HOURS` | No       | `24`    | Token lifetime in hours         |

See [Configuration](./config.md) for the full variable reference.
