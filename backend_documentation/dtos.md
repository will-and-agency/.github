# DTOs and Validation

**Location:** `src/dto/`

Data Transfer Objects (DTOs) are structs that define the **shape and validation rules** of incoming request bodies. They are separate from entity `Model` types, which represent database rows.

---

## Why DTOs Instead of Models

Using entity `Model` types directly as request bodies is problematic because:

- The client should not control fields like `id`, `created_at`, `created_by`, or `updated_at`.
- Input may need transformation before insertion (e.g., applying defaults, normalising data).
- Validation rules (required fields, length, date constraints) belong at the API boundary, not in the ORM layer.

DTOs make the accepted input surface explicit and safe.

---

## Validation Stack

Two crates work together for validation:

| Crate        | Role                                                              |
|--------------|-------------------------------------------------------------------|
| `validator`  | Defines validation rules via the `#[validate(...)]` attribute     |
| `axum-valid` | Runs the validator automatically during request deserialisation   |

In a handler, replacing `Json(payload)` with `Valid(Json(payload))` is all that is needed to activate validation. If validation fails, Axum returns a `422 Unprocessable Entity` response automatically — the handler body is never reached.

```rust
// Without validation
Json(payload): Json<CreateProjectDto>

// With validation (recommended)
Valid(Json(payload)): Valid<Json<CreateProjectDto>>
```

---

## Existing DTOs

### `CreateProjectDto` — `src/dto/projects.rs`

Used by `POST /projects/create`.

```rust
pub struct CreateProjectDto {
    pub company: String,              // required, min length 1
    pub name: String,                 // required, min length 1
    pub description: Option<String>,  // optional
    pub status: Option<String>,       // optional, defaults to "active" in handler
    pub settings: Option<Json>,       // optional, defaults to {} in handler
    pub starts_at: Option<DateTimeWithTimeZone>, // optional
    pub ends_at: DateTimeWithTimeZone,           // required
}
```

**Field-level validation:**

| Field     | Rule                                    |
|-----------|-----------------------------------------|
| `company` | `length(min = 1)` — cannot be empty     |
| `name`    | `length(min = 1)` — cannot be empty     |

**Schema-level validation** (applied to the whole struct):

```rust
#[validate(schema(function = "validate_project_dates"))]
```

The `validate_project_dates` function checks:

1. `ends_at` must be in the future (cannot be in the past).
2. If `starts_at` is provided, `ends_at` must be strictly after `starts_at`.

If either check fails, the request is rejected with a `422` before the handler body runs.

---

## Adding a New DTO

1. Create `src/dto/<domain>.rs`.
2. Declare the struct with `#[derive(Debug, Deserialize, Validate)]`.
3. Add `#[validate(...)]` attributes to fields as needed.
4. Add schema-level validation with `#[validate(schema(function = "..."))]` if cross-field rules are required.
5. Register the new module in `src/dto/mod.rs`: `pub mod <domain>;`.
6. Use `Valid(Json(payload)): Valid<Json<YourDto>>` in the handler.

### Validator Attribute Reference

```rust
#[validate(length(min = 1, max = 255))]
#[validate(email)]
#[validate(range(min = 0, max = 100))]
#[validate(url)]
#[validate(custom(function = "my_fn"))]
```

See the [validator crate docs](https://docs.rs/validator) for the full list.
