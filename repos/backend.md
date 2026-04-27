# MobileEtnoV2 — Backend Overview

> **Who this is for:** Anyone who needs to understand, run, or deploy the backend server.
> This document explains everything from scratch — no assumptions.

---

## What is this?

This is the **API server** for the MobileEtno platform. It is the brain of the whole system.
Both the web app (used by researchers/admins) and the mobile app (used by participants) talk to this server.

It handles:
- Logging users in
- Storing and serving projects, tasks, and survey questions
- Receiving answers from participants
- Real-time chat between participants and moderators (WebSocket)
- Push notifications
- File uploads (images, videos) via S3 cloud storage
- Sending invitation emails

---

## Tech Stack — What technologies are used?

| Technology | Version | What it does |
|---|---|---|
| **Rust** | 1.85+ | The programming language. Fast and safe. |
| **Axum** | 0.8.8 | The web framework. Handles HTTP routes. |
| **Tokio** | 1.50.0 | Makes the server handle many requests at once (async runtime). |
| **SeaORM** | 1.1.19 | Talks to the database. Turns Rust structs into SQL queries. |
| **PostgreSQL** | 16 | The database. Stores all the data. |
| **jsonwebtoken** | 9 | Creates and verifies JWT login tokens. |
| **bcrypt** | 0.15 | Hashes passwords so they're never stored in plain text. |
| **totp-rs** | 5.7.1 | Handles 2-factor authentication (like Google Authenticator). |
| **lettre** | 0.11 | Sends emails via SMTP. |
| **aws-sdk-s3** | 1 | Uploads and downloads files to S3-compatible cloud storage. |
| **tower-http** | 0.6 | Adds CORS headers, request tracing, and size limits. |

---

## Folder Structure — What does each folder do?

```
MobileEtnoV2_backend/
│
├── src/                        ← All the Rust source code lives here
│   ├── main.rs                 ← Entry point. Starts the server.
│   ├── lib.rs                  ← Exports modules so main.rs can use them.
│   ├── config.rs               ← Reads environment variables into a Config struct.
│   ├── db.rs                   ← Connects to PostgreSQL.
│   ├── state.rs                ← Shared app state (DB, config, storage, chat rooms).
│   ├── storage.rs              ← Abstraction over S3 file storage.
│   ├── errors.rs               ← Custom error types that turn into HTTP responses.
│   │
│   ├── handlers/               ← The functions that handle each HTTP request (16 files).
│   │   ├── auth.rs             ← Login, 2FA, password change handlers.
│   │   ├── project.rs          ← Create/read/update/delete projects.
│   │   ├── task.rs             ← Create/read/update/delete tasks.
│   │   ├── question.rs         ← Manage survey questions.
│   │   ├── answer.rs           ← Submit and read answers.
│   │   ├── attachment.rs       ← File upload/download logic.
│   │   ├── message.rs          ← Chat history and WebSocket chat.
│   │   ├── note.rs             ← Admin notes on answers.
│   │   ├── participant.rs      ← Participant management.
│   │   ├── notification.rs     ← Push notification management.
│   │   └── ...                 ← (moderator management, etc.)
│   │
│   ├── routes/                 ← Wires URLs to handlers (15 router files).
│   │   ├── auth.rs             ← /api/auth/* routes.
│   │   ├── project.rs          ← /api/projects/* routes.
│   │   └── ...                 ← (one file per domain area)
│   │
│   ├── middleware/             ← Code that runs before handlers check auth, roles.
│   │   ├── auth.rs             ← Extracts the JWT token from the request header.
│   │   └── rbac.rs             ← Role-based access control (who can do what).
│   │
│   ├── services/               ← Business logic (the rules of the app, 12 files).
│   │   ├── auth.rs             ← Password validation, token generation.
│   │   ├── email.rs            ← Builds and sends emails.
│   │   └── ...
│   │
│   ├── entities/               ← Database table models (18 tables, auto-generated).
│   └── dto/                    ← Data Transfer Objects (shapes of JSON in/out).
│
├── migration/                  ← Database schema changes over time.
│   └── src/                    ← 28 migration files (one per schema change).
│
├── tests/                      ← Integration tests.
│   ├── integration_test.rs     ← Test entry point.
│   └── auth/, answer/, ...     ← Test modules for each feature.
│
├── db/                         ← Docker configuration for the database.
│   ├── docker-compose.yml      ← Starts the dev PostgreSQL container.
│   └── docker-compose.test.yml ← Starts the test PostgreSQL container.
│
├── .github/workflows/          ← Automated CI/CD pipelines.
│   ├── rust.yml                ← Runs tests on every push.
│   └── docker.yml              ← Builds and publishes Docker image to GHCR.
│
├── Dockerfile                  ← Instructions to build a deployable Docker image.
├── docker-compose.yml          ← Starts the server + databases + mail for local dev.
├── Cargo.toml                  ← Rust project manifest (like package.json).
├── .env                        ← Your local environment variables (never commit this!).
└── .env.test                   ← Environment variables used during tests.
```

---

## How the Server Starts — Step by Step

When you run `cargo run`, here is exactly what happens in `src/main.rs`:

1. **Load .env file** — reads all environment variables from the `.env` file.
2. **Set up logging** — initialises structured logs controlled by `RUST_LOG`.
3. **Read config** — loads all env vars into a typed `Config` struct. If any required var is missing, the app panics immediately with a clear error message.
4. **Connect to PostgreSQL** — establishes a database connection pool.
5. **Connect to S3** — initialises the S3 storage client.
6. **Build AppState** — bundles the DB connection, config, storage, WebSocket chat rooms (DashMap), and notification channels into a single shared state object.
7. **Register all routes** — mounts every router under the `/api` prefix.
8. **Configure CORS** — sets allowed origins from the `CORS_ALLOWED_ORIGINS` env var.
9. **Add middleware** — attaches audit logging and request tracing.
10. **Bind and listen** — server starts listening on `0.0.0.0:3000` (or the port in `PORT` env var).

---

## Database — How Data is Stored

**Database:** PostgreSQL 16

**ORM:** SeaORM (Rust code ↔ database tables)

### The 17 Tables

| Table | What it stores |
|---|---|
| `accounts` | All user accounts (moderators and participants) |
| `accounts_projects` | Which moderator is assigned to which project |
| `projects` | Research projects |
| `tasks` | Tasks (chapters/sections) within a project |
| `questions` | Survey questions within a task |
| `question_options` | Answer choices for multiple-choice questions |
| `answers` | Participant answers to questions |
| `answer_options` | Which options a participant selected |
| `attachments` | Files (images, videos) linked to any entity |
| `messages` | Chat messages between a participant and moderators |
| `notes` | Admin-only notes on a participant's answer |
| `audit_logs` | A record of every important action in the system |
| `push_tokens` | Expo push tokens for mobile notifications |
| `participant_chat_dismissals` | Tracks which chat threads a participant has dismissed |
| `admin_chat_reads` | Tracks which messages an admin has read |
| `project_participant_ratings` | Star ratings given to participants per project |
| `notifications` | Persistent notification records |

### Migrations — How the Schema Changes Over Time

Migrations are in `migration/src/`. Each file adds, removes, or changes a table. They must be applied in order.

---

## Authentication — How Login Works

### User Roles

There are 4 roles in the system:

| Role | Who | What they can do |
|---|---|---|
| `root_moderator` | Platform owner (you) | Everything. Creates super moderators. |
| `super_moderator` | Senior researcher | Creates standard moderators. Manages projects. |
| `moderator` | Standard researcher | Manages assigned projects, tasks, questions. |
| `audience_user` | Research participant | Answers questions, chats with moderators. |

### Login Flow (Standard — No 2FA)

```
User sends email + password
        ↓
Server checks password against bcrypt hash in DB
        ↓
Server creates a JWT token (signed with JWT_SECRET)
        ↓
Server returns the full JWT to the user
        ↓
User stores the JWT and sends it as: Authorization: Bearer <token>
```

### Login Flow (With 2FA Enabled — Web App)

```
User sends email + password
        ↓
Server validates password ✓
        ↓
Server returns a TEMPORARY JWT (valid 5 minutes, marked step="pending_2fa")
        ↓
User opens their authenticator app and gets a 6-digit code
        ↓
User sends: POST /auth/verify-2fa  {code: "123456"}  with temp token
        ↓
Server verifies the TOTP code ✓
        ↓
Server returns a FULL JWT token
```

### JWT Token Structure

```json
{
  "sub": "uuid-of-the-user",
  "role": "moderator",
  "exp": 1234567890,
  "step": null
}
```
- `sub` — the user's UUID
- `role` — determines what they can access
- `exp` — when the token expires (default: 24 hours, set by `JWT_EXPIRY_HOURS`)
- `step` — `"pending_2fa"` for temporary tokens, `null` for full tokens

### Password Rules

- At least 8 characters
- At least one uppercase letter
- At least one lowercase letter
- At least one number

---

## All API Routes

All routes are prefixed with `/api`.

### Auth (`/api/auth`)

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/auth/web/login` | Anyone | Web login (returns temp or full JWT) |
| POST | `/auth/app/login` | Anyone | App login (no 2FA, returns full JWT) |
| POST | `/auth/change-initial-password` | New accounts | Sets a permanent password on first login |
| POST | `/auth/setup-2fa` | Moderators | Generates a TOTP secret + QR code |
| POST | `/auth/verify-2fa` | Temp token holders | Verifies TOTP code, returns full JWT |

### Users (`/api/me`, `/api/users`)

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/me` | Any authenticated | Returns the current user's profile |
| GET | `/users` | Root only | Returns all accounts |
| GET | `/user/{id}` | Root/Super | Returns one account |

### Projects (`/api/projects`)

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/projects/create` | Super/Root | Creates a new project |
| GET | `/projects/get` | Moderators | Lists projects (filtered by assignment for standard mods) |
| GET | `/projects/{id}/get` | Moderators | Gets one project's details |
| PUT | `/projects/update/{id}` | Super/Root | Updates a project |
| DELETE | `/projects/delete/{id}` | Super/Root | Deletes a project |

### Tasks (`/api/projects/{projectId}/tasks`)

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/tasks/create` | Moderators | Creates a task inside a project |
| GET | `/tasks/get` | Moderators + Participants | Lists tasks |
| GET | `/tasks/{taskId}/get` | Moderators | Gets one task |
| PUT | `/tasks/{taskId}/update` | Moderators | Updates a task |
| DELETE | `/tasks/{taskId}/delete` | Moderators | Deletes a task |

### Questions (`/api/projects/{projectId}/tasks/{taskId}/questions`)

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/questions/create` | Moderators | Creates a question |
| GET | `/questions/get` | Moderators + Participants | Lists questions |
| PUT | `/questions/{id}/update` | Moderators | Updates a question |
| DELETE | `/questions/{id}/delete` | Moderators | Deletes a question |

### Question Options (Multiple Choice)

Under `/questions/{questionId}/options`:

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/options` | Moderators + Participants | Lists options for a question |
| POST | `/options` | Moderators | Creates an option |
| PUT | `/options/{optionId}` | Moderators | Updates an option |
| DELETE | `/options/{optionId}` | Moderators | Deletes an option |

### Answers

Under `/questions/{questionId}/answers`:

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/answers/create` | Participants | Submits an answer |
| GET | `/answers/my` | Participants | Gets the participant's own answer |
| PUT | `/answers/{id}/update` | Participants | Updates their answer |
| GET | `/answers/get` | Moderators | Lists all answers (admin view) |
| GET | `/answers/{id}/get` | Moderators | Gets one specific answer |

### Attachments (File Uploads)

Uploads use a 3-step process: **init → upload to S3 → confirm**

| Step | Endpoint | What happens |
|---|---|---|
| 1. Init | `POST /attachments/init` | Server returns a pre-signed S3 URL |
| 2. Upload | `PUT <s3-url>` (direct to S3) | Client uploads file directly to S3 |
| 3. Confirm | `POST /attachments/{id}/confirm` | Server records the upload is complete |

Attachments can be linked to: projects, tasks, questions, answers, and question options.

### Participants (`/api/projects/{projectId}/participants`)

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/participants/upload-csv` | Moderators | Creates participant accounts from a CSV file |
| POST | `/participants/send-invites` | Moderators | Sends invitation emails to participants |
| GET | `/participants` | Moderators | Lists all participants in a project |
| GET | `/participants/{id}` | Moderators | Gets one participant's details |
| POST | `/participants/{id}/deactivate` | Moderators | Disables a participant |
| POST | `/participants/{id}/rate` | Moderators | Rates a participant (star rating) |
| GET | `/participants/{id}/rate` | Moderators | Gets a participant's rating |

### Chat / Messages

| Method | Path | Who | What |
|---|---|---|---|
| GET (WebSocket) | `/ws/projects/{p}/participants/{u}/questions/{q}/get` | Auth | Real-time chat connection |
| GET | `/projects/{p}/participants/{u}/questions/{q}/history` | Auth | Loads chat history |
| GET | `/projects/{p}/participants/{u}/unread` | Moderators | Gets unread message count |

### Notifications

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/push-tokens` | Participants | Registers Expo push token |
| DELETE | `/push-tokens` | Participants | Removes push token |
| GET | `/notifications` | Moderators | Lists moderator notifications |
| POST | `/notifications/mark-read` | Moderators | Marks notification as read |
| GET | `/participant/notifications` | Participants | Lists participant notifications |
| POST | `/participant/notifications/dismiss` | Participants | Dismisses a notification |
| GET (WebSocket) | `/ws/notifications` | Auth | Real-time notification stream |

### Notes (`/api/projects/{projectId}/answers/{answerId}/notes`)

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/notes/create` | Moderators | Adds a private note on an answer |
| GET | `/notes/get` | Moderators | Lists notes on an answer |
| PUT | `/notes/{id}/update` | Moderators | Updates a note |
| DELETE | `/notes/{id}/delete` | Moderators | Deletes a note |

### Moderator Management

| Method | Path | Who | What |
|---|---|---|---|
| POST | `/moderators/super/create` | Root | Creates a super moderator |
| POST | `/moderators/standard/create` | Super/Root | Creates a standard moderator |
| PATCH | `/moderators/super/{id}/update` | Root | Updates super mod details |
| PATCH | `/moderators/standard/{id}/update` | Root | Updates standard mod details |
| PATCH | `/moderators/super/{id}/status-update` | Root | Suspends/activates super mod |
| PATCH | `/accounts/{id}/status-update` | Root | Suspends/activates standard mod |
| POST | `/moderators/super/{id}/delete/initiate` | Root | Starts 2-step deletion |
| POST | `/moderators/super/{id}/delete/confirm` | Root | Confirms deletion |
| POST | `/moderators/standard/{id}/assign-project` | Super/Root | Assigns mod to a project |
| GET | `/moderators/standard/{id}/projects` | Super/Root | Lists mod's projects |
| DELETE | `/moderators/standard/{id}/deassign-project/{pid}` | Super/Root | Removes mod from a project |

---

## Environment Variables — What You Must Configure

Copy `.env` and fill in these values before running:

```env
# ── DATABASE ─────────────────────────────────────────────────────────
# Full PostgreSQL connection string
DATABASE_URL=postgresql://user:password@127.0.0.1:5432/db-dev

# ── SECURITY ─────────────────────────────────────────────────────────
# Long random string used to sign JWT tokens. Keep this secret!
# Generate one: openssl rand -base64 48
JWT_SECRET=your-long-random-secret

# How many hours a login token stays valid (24 = one day)
JWT_EXPIRY_HOURS=24

# 64 hex characters (32 bytes) used to encrypt TOTP secrets
# Generate one: openssl rand -hex 32
TOTP_ENCRYPTION_KEY=0000000000000000000000000000000000000000000000000000000000000000

# ── SERVER ───────────────────────────────────────────────────────────
PORT=3000

# Which web origins are allowed to call this API
# For development: *
# For production: https://yourdomain.com,https://www.yourdomain.com
CORS_ALLOWED_ORIGINS=*

# ── EMAIL (SMTP) ─────────────────────────────────────────────────────
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
SMTP_FROM=your-email@gmail.com

# ── FILE STORAGE (S3-compatible) ─────────────────────────────────────
S3_ENDPOINT=https://s3.eu-west-par.io.cloud.ovh.net/
S3_REGION=eu-west-par
S3_BUCKET=your-bucket-name
S3_ACCESS_KEY=your-access-key
S3_SECRET_KEY=your-secret-key
```

---

## How to Run Locally — Step by Step

### Prerequisites

1. Install Rust: go to https://rustup.rs and follow the instructions.
2. Install Docker Desktop.
3. Install the SeaORM CLI:
   ```bash
   cargo install sea-orm-cli
   ```

### Steps

```bash
# Step 1: Go into the project folder
cd MobileEtnoV2_backend

# Step 2: Start the PostgreSQL database (runs in Docker)
docker compose up -d db

# Step 3: Make sure your .env file is filled in (copy from .env.example or .env and edit)

# Step 4: Apply all database migrations (creates all the tables)
sea-orm-cli migrate up

# Step 5: Run the server
cargo run
```

The server will start and print something like:
```
Listening on 0.0.0.0:3000
```

You can now make API calls to `http://localhost:3000/api/...`.

### Useful Development Commands

```bash
# Run the full test suite
cargo test

# Format your code (run before committing)
cargo fmt --all

# Check for code quality issues
cargo clippy

# Run with debug-level logs
RUST_LOG=debug cargo run

# Run only one test module
cargo test --test integration_test auth::
```

---

## Docker & Docker Compose

### For Local Development

The `docker-compose.yml` in the root starts these services:

| Service | Port | What it is |
|---|---|---|
| `db` | 5432 | PostgreSQL 16 — the main development database |
| `db_test` | 5433 | PostgreSQL 16 — the test database (data deleted after tests) |
| `mailpit` | 1025 (SMTP), 8025 (UI) | Catches emails locally so you can inspect them in a browser |

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Stop and delete all data (fresh start)
docker compose down -v
```

### The Dockerfile (for Production)

The `Dockerfile` builds the server in two stages:

**Stage 1 — Builder**
- Uses `rust:1.91-slim`
- Compiles the Rust code into a single binary (`target/release/backend`)

**Stage 2 — Runtime**
- Uses `debian:bookworm-slim` (very small image)
- Copies only the compiled binary
- Exposes port 3000
- Runs `/app/backend`

The final image is small because it doesn't include the Rust toolchain.

---

## CI/CD — Automated Pipelines

These live in `.github/workflows/`.

### `rust.yml` — Runs on Every Push

**Triggers:** Push to `main` or `develop`, pull requests to those branches.

**What it does:**
1. Starts a PostgreSQL and Mailpit service container in GitHub Actions.
2. Installs the Rust toolchain with `rustfmt` and `clippy`.
3. Caches Rust dependencies (so builds are faster).
4. Checks code formatting: `cargo fmt --check`
5. Runs linter: `cargo clippy`
6. Builds: `cargo build --verbose`
7. Runs all tests: `cargo test --verbose`

If any step fails, the PR is blocked.

### `docker.yml` — Runs on Push to `main`

**What it does:**
1. Builds the Docker image using the multi-stage `Dockerfile`.
2. Logs into the GitHub Container Registry (GHCR) using the GitHub token.
3. Pushes the image with two tags:
   - `latest` — always points to the newest build on main.
   - `sha-<short-hash>` — immutable tag tied to a specific commit.

The image is stored at: `ghcr.io/<github-org>/mobileetno-backend`

---

## Tests

Tests live in `tests/`. They are **integration tests** — they spin up the real server and hit real endpoints.

| Test folder | What is tested |
|---|---|
| `auth/` | Login, 2FA, TOTP codes, RBAC, account lockout |
| `answer/` | Submitting and updating answers |
| `question/` | Creating and managing questions |
| `attachment/` | File upload flow |
| `project/` | Project CRUD |
| `task/` | Task CRUD |
| `participants/` | CSV upload, invitations |
| `messages/` | WebSocket chat |
| `note/` | Notes on answers |
| `email/` | Email sending |

Tests use the database on port **5433** (defined in `.env.test`). That database is ephemeral — it resets between test runs.

---

## Deployment Checklist

Use this when deploying to a server for the first time.

### Infrastructure You Need

- [ ] A server running Linux (Ubuntu 22.04+ recommended)
- [ ] PostgreSQL 16 accessible from the server
- [ ] S3-compatible object storage (OVHcloud, MinIO, AWS S3, etc.)
- [ ] An SMTP server or service (Gmail, Resend, SendGrid, etc.)
- [ ] A reverse proxy (Nginx or Caddy) for HTTPS/TLS

### Environment Variables — Must Set Before Starting

- [ ] `DATABASE_URL` — PostgreSQL connection string
- [ ] `JWT_SECRET` — strong random string (generate: `openssl rand -base64 48`)
- [ ] `JWT_EXPIRY_HOURS` — e.g., `24`
- [ ] `TOTP_ENCRYPTION_KEY` — 64 hex chars (generate: `openssl rand -hex 32`)
- [ ] `PORT` — e.g., `3000`
- [ ] `CORS_ALLOWED_ORIGINS` — your production domain(s), comma-separated
- [ ] `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`
- [ ] `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`

### Steps to Deploy

```bash
# 1. Pull the Docker image
docker pull ghcr.io/<your-org>/mobileetno-backend:latest

# 2. Run the database migrations
# (do this before starting the server for the first time, or after updates)
docker run --rm \
  --env-file .env.production \
  ghcr.io/<your-org>/mobileetno-backend:latest \
  /app/migration up

# 3. Start the server
docker run -d \
  --name mobileetno-backend \
  --restart unless-stopped \
  --env-file .env.production \
  -p 3000:3000 \
  ghcr.io/<your-org>/mobileetno-backend:latest
```

### Important Notes for Production

- **HTTPS:** The app does NOT handle TLS itself. Put Nginx or Caddy in front of it.
- **WebSockets:** Nginx must be configured to proxy WebSocket connections (`proxy_http_version 1.1`, `Upgrade`, `Connection` headers).
- **CORS:** Set `CORS_ALLOWED_ORIGINS` to exactly your frontend domain. Do not use `*` in production.
- **Data persistence:** The PostgreSQL `postgres_data` Docker volume must be backed up regularly.
- **Multiple instances:** If you run more than one backend instance behind a load balancer, you must use **sticky sessions** because WebSocket chat rooms are stored in memory (DashMap). They do not sync across instances.
- **Log level:** Set `RUST_LOG=info` for production (not `debug` — too verbose).
