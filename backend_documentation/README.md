# Backend Documentation

This is the reference documentation for the **MobileEtnoV2** backend — a REST API built with Rust, Axum, and SeaORM backed by PostgreSQL.

## Tech Stack

| Concern          | Library / Tool                          |
|------------------|-----------------------------------------|
| HTTP Framework   | [Axum 0.8](https://docs.rs/axum)        |
| ORM              | [SeaORM 1.1](https://www.sea-ql.org/SeaORM/) |
| Database         | PostgreSQL                              |
| Auth             | JWT (`jsonwebtoken`) + bcrypt           |
| Validation       | `validator` + `axum-valid`              |
| Async runtime    | Tokio                                   |
| Serialisation    | Serde (JSON)                            |

---

## Documentation Index

### Architecture
- [Architecture Overview](./architecture/overview.md) — layers, request lifecycle, module map
- [Database Schema](./architecture/database-schema.md) — entity relationships and field reference

### Layers
- [Routes](./layers/routes.md) — URL-to-handler mapping
- [Handlers](./layers/handlers.md) — request/response logic
- [Services](./layers/services.md) — reusable business logic
- [Entities](./layers/entities.md) — SeaORM models and relations

### Cross-Cutting Concerns
- [Authentication & Auth Middleware](./auth.md) — JWT generation, verification, `AuthUser` extractor
- [DTOs & Validation](./dtos.md) — data transfer objects and input validation
- [Error Handling](./errors.md) — `AppError` enum and HTTP mapping
- [Configuration](./config.md) — environment variables and `Config` struct
