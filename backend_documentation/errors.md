# Error Handling

**Location:** `src/errors.rs`

All handler errors flow through a single `AppError` enum. Axum converts it to an HTTP response via the `IntoResponse` trait implementation.

---

## `AppError` Enum

```rust
pub enum AppError {
    InternalServerError(String),
    DatabaseError(String),
    BadRequest(String),
    NotFound(String),
}
```

Each variant carries a `String` message that becomes the response body.

---

## HTTP Mapping

| Variant                | HTTP Status                    |
|------------------------|-------------------------------|
| `InternalServerError`  | `500 Internal Server Error`    |
| `DatabaseError`        | `500 Internal Server Error`    |
| `BadRequest`           | `400 Bad Request`              |
| `NotFound`             | `404 Not Found`                |

The response body is the plain-text message string.

---

## Usage in Handlers

### Explicit construction

```rust
if existing.is_some() {
    return Err(AppError::BadRequest("Email already in use".to_string()));
}

Account::find_by_id(id)
    .one(&state.db)
    .await?
    .ok_or_else(|| AppError::NotFound("User not found".to_string()))?;
```

### Via `map_err`

```rust
hash_password(&body.password)
    .map_err(|e| AppError::InternalServerError(e.to_string()))?;
```

### Via the `?` operator (SeaORM errors)

`AppError` implements `From<sea_orm::DbErr>`, so any SeaORM query that returns `DbErr` can be propagated directly:

```rust
// Automatically converted to AppError::DatabaseError
let projects = projects::Entity::find().all(&state.db).await?;
```

This keeps handler code concise and avoids boilerplate `map_err` calls for database operations.

---

## Adding a New Variant

1. Add the variant to the `AppError` enum.
2. Add a matching arm in the `IntoResponse` implementation with the desired `StatusCode`.
3. Optionally implement `From<SomeExternalError> for AppError` if the new variant wraps a third-party error type.

---

## Validation Errors

Input validation errors (from `axum-valid` / `validator`) are handled automatically by the `axum-valid` crate and return a `422 Unprocessable Entity`. They bypass `AppError` entirely and do not need to be mapped manually.
