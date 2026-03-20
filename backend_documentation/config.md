# Configuration

**Location:** `src/config.rs`

All configuration is read from environment variables at startup. The `Config` struct is populated once in `main.rs` and then shared across the application via `AppState`.

---

## Environment Variables

| Variable           | Required | Default | Description                                     |
|--------------------|----------|---------|-------------------------------------------------|
| `DATABASE_URL`     | Yes      | —       | PostgreSQL connection string                    |
| `JWT_SECRET`       | Yes      | —       | Secret key used to sign and verify JWTs (HS256) |
| `JWT_EXPIRY_HOURS` | No       | `24`    | How many hours a generated token is valid for   |
| `PORT`             | No       | `3000`  | TCP port the HTTP server binds to               |

### Required Variables

If a required variable is missing, the application **panics immediately at startup** with a clear message:

```
DATABASE_URL environment variable is required
```

This is intentional — failing fast at boot is safer than failing silently on the first request.

### Optional Variables

Optional variables have hardcoded defaults. If `PORT` or `JWT_EXPIRY_HOURS` are not set, the defaults are used.

---

## `Config` Struct

```rust
pub struct Config {
    pub database_url: String,
    pub jwt_secret: String,
    pub jwt_expiry_hours: i64,
    pub port: u16,
}
```

Populated via `Config::from_env()` in `main.rs`:

```rust
let config = config::Config::from_env();
```

---

## Development Setup

For local development, place a `.env` file in the project root. The `dotenvy` crate loads it automatically on startup (this is disabled in production where real environment variables are used instead).

Example `.env`:

```env
DATABASE_URL=postgres://user:password@localhost:5432/mobileetno
JWT_SECRET=a-very-long-random-secret-string
JWT_EXPIRY_HOURS=24
PORT=3000
```

`.env` is gitignored and should never be committed.

---

## Adding a New Config Value

1. Add the field to the `Config` struct in `src/config.rs`.
2. Populate it in `Config::from_env()` using either `required("VAR_NAME")` or `optional("VAR_NAME", "default")`.
3. Document it in this file's variable table above.

Helper functions in `config.rs`:

```rust
// Panics if the variable is missing
fn required(name: &str) -> String { ... }

// Returns the default value if the variable is missing
fn optional(name: &str, default: &str) -> String { ... }
```

---

## Planned / Commented-Out Config

The following config fields are stubbed in `config.rs` for future use:

```rust
// S3 / Object Storage
// pub s3_endpoint: String
// pub s3_access_key: String
// pub s3_secret_key: String
// pub s3_bucket_uploads: String
// pub s3_region: String

// Redis (for job queue)
// pub redis_url: String
```

Uncomment and wire these up when file upload or background job functionality is added.
