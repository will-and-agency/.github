# Services Layer

**Location:** `src/services/`

Services contain **pure, reusable business logic** that does not belong in a single handler. They are plain Rust functions (sync or async) with no Axum dependency — they do not return HTTP responses and have no knowledge of request/response types.

---

## When to Use a Service vs. a Handler

For auxilary functions that do business logic. These should not go to handler.

| Logic type                                    | Where it lives  |
|-----------------------------------------------|-----------------|
| Hashing a password                            | `services/auth` |
| Generating / verifying a JWT                  | `services/auth` |
| Database query that is called from one handler only | handler itself |
| Business rule reused across multiple handlers | `services/`     |
| HTTP error mapping                            | `errors.rs`     |

---

## `src/services/auth.rs`

The auth service owns all cryptographic operations and the JWT lifecycle.

### `Claims`

The payload embedded inside every JWT.

```rust
pub struct Claims {
    pub sub: Uuid,    // the account's UUID (subject)
    pub role: String, // "admin" | "user" | "moderator"
    pub exp: usize,   // Unix timestamp — token expiry
}
```

Helper methods:

| Method             | Description                                      |
|--------------------|--------------------------------------------------|
| `has_role(&str)`   | Returns `true` if `self.role == role`            |
| `is_expired()`     | Returns `true` if the current time exceeds `exp` |

### `Roles`

A constants namespace to avoid magic strings in handler code.

```rust
Roles::ADMIN      // "admin"
Roles::USER       // "user"
Roles::MODERATOR  // "moderator"
```

### Functions

#### `hash_password(password: &str) -> Result<String, BcryptError>`

Hashes a plaintext password using bcrypt at the default cost factor. The result is stored in `accounts.password_hash`.

#### `verify_password(password: &str, hash: &str) -> Result<bool, BcryptError>`

Compares a plaintext password against a stored bcrypt hash. Returns `Ok(true)` on match.

#### `generate_token(account_id, role, secret, expiry_hours) -> Result<String, JwtError>`

Creates a signed HS256 JWT with the provided `Claims`. The expiry is calculated as `now + expiry_hours`.

#### `verify_token(token: &str, secret: &str) -> Result<Claims, JwtError>`

Decodes and validates a JWT string. Returns the embedded `Claims` on success. Fails if the signature is invalid or the token is expired.

---

## `src/services/user.rs` / `src/services/project.rs`

These modules are currently empty placeholders. As the application grows, database queries that need to be shared across multiple handlers (e.g., "find user by email", "check project ownership") should be extracted here.
