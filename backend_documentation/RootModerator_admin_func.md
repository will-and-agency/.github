# feature/8-rootmoderator — Complete Branch Documentation

This document covers everything implemented on the `feature/8-rootmoderator` branch: authentication flows, account management APIs, role system, security features, email delivery, and test infrastructure.

---

## Table of Contents

1. [Role System](#1-role-system)
2. [Authentication Flows](#2-authentication-flows)
   - [First Login (New Account)](#21-first-login-new-account)
   - [Normal Login (Returning User)](#22-normal-login-returning-user)
3. [Token Types](#3-token-types)
4. [API Reference — Auth](#4-api-reference--auth)
5. [API Reference — Super Moderator Management](#5-api-reference--super-moderator-management)
6. [API Reference — Standard Moderator Management](#6-api-reference--standard-moderator-management)
7. [RBAC Permission Matrix](#7-rbac-permission-matrix)
8. [Security Features](#8-security-features)
9. [Email / OTP System](#9-email--otp-system)
10. [Environment Configuration](#10-environment-configuration)
11. [Test Infrastructure](#11-test-infrastructure)
12. [Running Tests](#12-running-tests)
13. [Architecture Notes](#13-architecture-notes)

---

## 1. Role System

There are three roles stored in the `accounts.role` column:

| Role string (exact DB value) | Who it is |
|---|---|
| `root_moderator` | The top-level admin. Created manually / via seed. Has all permissions. |
| `super_moderator` | A manager-level moderator. Created by Root. Can create standard moderators. |
| `moderator` | A standard moderator. Created by Root or Super. Lowest privilege. |

**Important**: the role strings in the database are `root_moderator`, `super_moderator`, and `moderator` — not `root`, `super`, or `admin`. The RBAC middleware checks these exact strings.

---

## 2. Authentication Flows

All new accounts receive a temporary 6-digit OTP code via email, which serves as their initial password. They must change it on first login before accessing anything.

### 2.1 First Login (New Account)

This is the flow for any account that was just created (super moderator or standard moderator).

```
Step 1: POST /auth/login
  → receives requires_password_change: true
  → receives a temp_token (pending_password_change)

Step 2: POST /auth/change-initial-password
  → sends the new password
  → receives a new temp_token (pending_2fa) + setup_required: true

Step 3: POST /auth/setup-2fa
  → receives secret + uri (QR code)
  → user scans the QR with an authenticator app (e.g., Google Authenticator)

Step 4: POST /auth/verify-2fa
  → sends the 6-digit code from the authenticator app
  → receives full_token (the real JWT for all API calls)
  → This also marks totp_enabled = true for the account (TOTP is now activated)
```

**Step 1 — Login request:**
```json
POST /auth/login
{
  "email": "jane@example.com",
  "password": "123456"   // the OTP code from the email
}
```
**Step 1 — Login response (new account):**
```json
{
  "temp_token": "eyJ...",
  "requires_password_change": true
}
```

**Step 2 — Change password:**
```json
POST /auth/change-initial-password
Authorization: Bearer <temp_token from step 1>
{
  "new_password": "MyNewPass1"
}
```
Password rules: minimum 8 characters, at least one uppercase, one lowercase, one number.

**Step 2 — Response:**
```json
{
  "temp_token": "eyJ...",   // new temp_token, step = pending_2fa
  "setup_required": true
}
```

**Step 3 — Setup TOTP (scan QR code):**
```json
POST /auth/setup-2fa
Authorization: Bearer <temp_token from step 2>
// No request body needed
```
**Step 3 — Response:**
```json
{
  "secret": "JBSWY3DPEHPK3PXP",   // base32 secret for manual entry
  "uri": "otpauth://totp/MobileEtno:jane@example.com?secret=JBSWY3DPEHPK3PXP&issuer=MobileEtno"
}
```
The `uri` is what authenticator apps scan as a QR code. The `secret` is for manual entry.

**Step 4 — Verify TOTP code:**
```json
POST /auth/verify-2fa
Authorization: Bearer <temp_token from step 2>
{
  "code": "482910"   // current 6-digit code from authenticator app
}
```
**Step 4 — Response:**
```json
{
  "full_token": "eyJ..."   // the real JWT — use this for all future API calls
}
```

---

### 2.2 Normal Login (Returning User)

For an account that has already completed first login and has TOTP configured:

```
Step 1: POST /auth/login
  → receives requires_2fa: true
  → receives a temp_token (pending_2fa)

Step 2: POST /auth/verify-2fa
  → sends the current 6-digit code from the authenticator app
  → receives full_token
```

**Step 1 — Login:**
```json
POST /auth/login
{
  "email": "jane@example.com",
  "password": "MyNewPass1"
}
```
**Step 1 — Response:**
```json
{
  "temp_token": "eyJ...",
  "requires_2fa": true
}
```

**Step 2 — Verify TOTP:**
```json
POST /auth/verify-2fa
Authorization: Bearer <temp_token>
{
  "code": "482910"
}
```
**Step 2 — Response:**
```json
{
  "full_token": "eyJ..."
}
```

---

## 3. Token Types

Every JWT contains these fields:

| Field | Description |
|---|---|
| `sub` | UUID of the account |
| `role` | Role string (`root_moderator`, `super_moderator`, `moderator`) |
| `exp` | Expiry timestamp (Unix seconds) |
| `step` | Optional. Present only in temp tokens. |

There are two token types:

### Temp Token (`step` is present)
- Created by `POST /auth/login` or `POST /auth/change-initial-password`
- Expires in **5 minutes**
- `step` is either `"pending_2fa"` or `"pending_password_change"`
- Can **only** be used on `/auth/setup-2fa`, `/auth/verify-2fa`, `/auth/change-initial-password`
- **Cannot** access any protected resource routes — RBAC middleware rejects it with 403

### Full Token (`step` is absent)
- Created by `POST /auth/verify-2fa`
- Expires based on `JWT_EXPIRY_HOURS` env var (default: 24 hours)
- `step` field is entirely absent from the token (not null — not there at all)
- Required by all protected routes (account management, etc.)

---

## 4. API Reference — Auth

All auth routes are under no prefix. Request body size is limited to 4 KB.

### POST /auth/login

Validates email + password. Returns a temp token.

**Request:**
```json
{ "email": "string", "password": "string" }
```

**Possible responses:**

| Scenario | Response body |
|---|---|
| First-ever login (OTP as password, must change) | `{ "temp_token": "...", "requires_password_change": true }` |
| TOTP already configured | `{ "temp_token": "...", "requires_2fa": true }` |
| TOTP not yet set up | `{ "temp_token": "...", "setup_required": true }` |

**Error responses:**
- `401` — wrong email, wrong password, inactive account, or account is locked
- `500` — DB failure

**Security details:**
- Returns 401 (not 404) for unknown email — prevents email enumeration
- After 5 wrong passwords → account is locked for 5 minutes
- Successful login resets failed attempt counter and removes lock

---

### POST /auth/change-initial-password

Only callable with a `pending_password_change` temp token. Sets a new password and returns a new `pending_2fa` temp token.

**Headers:** `Authorization: Bearer <temp_token (pending_password_change)>`

**Request:**
```json
{ "new_password": "string" }
```

**Password rules:** ≥ 8 chars, at least 1 uppercase, 1 lowercase, 1 number.

**Response:**
```json
{
  "temp_token": "...",
  "requires_2fa": true     // if TOTP already set up
  // OR
  "setup_required": true   // if TOTP not yet set up
}
```

---

### POST /auth/setup-2fa

Only callable with a `pending_2fa` temp token. Generates a new TOTP secret, stores it encrypted, and returns the secret + URI for the authenticator app. Does NOT enable TOTP yet — that happens when `verify-2fa` succeeds.

**Headers:** `Authorization: Bearer <temp_token (pending_2fa)>`

**No request body.**

**Response:**
```json
{
  "secret": "BASE32SECRET",
  "uri": "otpauth://totp/..."
}
```

---

### POST /auth/verify-2fa

Validates a 6-digit TOTP code. If TOTP was not yet enabled, enables it. Returns a full JWT.

**Headers:** `Authorization: Bearer <temp_token (pending_2fa)>`

**Request:**
```json
{ "code": "123456" }
```

**Response:**
```json
{ "full_token": "eyJ..." }
```

**Error responses:**
- `401` — wrong code, replayed code (same 30-second window used twice), or full token passed instead of temp token
- `400` — TOTP not configured yet (call setup-2fa first)

---

## 5. API Reference — Super Moderator Management

All super moderator routes require **Root only** (`root_moderator` role).

Use the full token in `Authorization: Bearer <full_token>` for all these.

---

### POST /moderators/super

Create a new Super Moderator account. Sends an OTP code to their email as their initial password.

**Requires:** Root

**Request:**
```json
{
  "email": "jane@example.com",
  "first_name": "Jane",
  "last_name": "Doe"
}
```

**Response (200):**
```json
{
  "message": "SuperModerator created successfully",
  "account_id": "uuid-here"
}
```

**What happens internally:**
1. A random 6-digit OTP code is generated
2. The account is created in the DB with role `super_moderator`, the OTP as a hashed password, and `metadata.must_change_password = true`
3. An email is sent to their address with the OTP code
4. On first login, they must change the password before doing anything else

---

### PATCH /moderators/super/{id}/status

Activate or suspend a super moderator account.

**Requires:** Root

**Path param:** `id` — UUID of the super moderator

**Request:**
```json
{ "is_active": false }   // false = suspend, true = activate
```

**Response (200):**
```json
{
  "message": "Account suspended successfully",
  "account_id": "uuid-here"
}
```

---

### PATCH /moderators/super/{id}

Edit a super moderator's personal details. All fields are optional — only send what you want to change.

**Requires:** Root

**Path param:** `id` — UUID of the super moderator

**Request:**
```json
{
  "first_name": "Jane",    // optional
  "last_name": "Smith",    // optional
  "email": "new@email.com" // optional
}
```

**Response (200):**
```json
{
  "message": "Account updated successfully",
  "account_id": "uuid-here"
}
```

---

### POST /moderators/super/{id}/delete/initiate

**Step 1 of 2** for safe deletion. Stores a one-time confirmation UUID in the account's metadata.

**Requires:** Root

**Path param:** `id` — UUID of the super moderator

**No request body.**

**Response (200):**
```json
{
  "confirmation_id": "some-uuid",
  "message": "Deletion initiated. Please confirm with the provided ID."
}
```

Save the `confirmation_id` — you need it for step 2.

---

### POST /moderators/super/{id}/delete/confirm

**Step 2 of 2** for safe deletion. Permanently deletes the account. The `confirmation_id` must match what was returned by initiate.

**Requires:** Root

**Path param:** `id` — UUID of the super moderator

**Request:**
```json
{ "confirmation_id": "some-uuid-from-initiate" }
```

**Response (200):**
```json
{
  "message": "Account deleted successfully",
  "account_id": "uuid-here"
}
```

**Error:** `400` if the confirmation ID doesn't match what's stored.

---

## 6. API Reference — Standard Moderator Management

---

### POST /moderators/standard

Create a new Standard Moderator. Sends an OTP code to their email.

**Requires:** Super or Root (both roles can create standard moderators)

**Request:**
```json
{
  "email": "mod@example.com",
  "first_name": "John",
  "last_name": "Doe"
}
```

**Response (200):**
```json
{
  "message": "Moderator created successfully",
  "account_id": "uuid-here"
}
```

**Error:** `400` if the email is already registered.

---

### PATCH /accounts/{id}/status

Activate or suspend a standard moderator.

**Requires:** Root only

**Path param:** `id` — UUID of the standard moderator

**Request:**
```json
{ "is_active": false }
```

**Response (200):**
```json
{
  "message": "Account suspended successfully",
  "account_id": "uuid-here"
}
```

---

### PATCH /moderators/standard/{id}

Edit a standard moderator's personal details. All fields optional.

**Requires:** Root only

**Path param:** `id` — UUID of the standard moderator

**Request:**
```json
{
  "first_name": "John",
  "last_name": "Updated",
  "email": "newemail@example.com"
}
```

**Response (200):**
```json
{
  "message": "Account updated successfully",
  "account_id": "uuid-here"
}
```

---

### POST /accounts/{id}/delete/initiate

Step 1 of safe deletion for a standard moderator.

**Requires:** Root only

**No request body.**

**Response (200):**
```json
{
  "confirmation_id": "some-uuid",
  "message": "Deletion initiated. Please confirm with the provided ID."
}
```

---

### POST /accounts/{id}/delete/confirm

Step 2 of safe deletion. Permanently deletes the account.

**Requires:** Root only

**Request:**
```json
{ "confirmation_id": "some-uuid-from-initiate" }
```

**Response (200):**
```json
{
  "message": "Account deleted successfully",
  "account_id": "uuid-here"
}
```

---

## 7. RBAC Permission Matrix

| Action | Root | Super | Moderator |
|---|:---:|:---:|:---:|
| Create super moderator | ✅ | ❌ | ❌ |
| Suspend/activate super moderator | ✅ | ❌ | ❌ |
| Edit super moderator details | ✅ | ❌ | ❌ |
| Delete super moderator | ✅ | ❌ | ❌ |
| Create standard moderator | ✅ | ✅ | ❌ |
| Suspend/activate standard moderator | ✅ | ❌ | ❌ |
| Edit standard moderator details | ✅ | ❌ | ❌ |
| Delete standard moderator | ✅ | ❌ | ❌ |

The RBAC is enforced via Axum extractor structs in `src/middleware/rbac.rs`:

- **`RequireRoot`** — accepts only `role == "root_moderator"`
- **`RequireSuper`** — accepts `"root_moderator"` or `"super_moderator"`
- **`RequireModerator`** — accepts all three roles

If the role check fails → `403 Forbidden`. If the token is a temp token (has a `step` field) → `403 Forbidden`. If no token → `401 Unauthorized`.

---

## 8. Security Features

### Password Lockout
After 5 consecutive wrong passwords on the same account, that account is locked for 5 minutes. The `failed_attempts` and `locked_until` columns track this. A successful login resets both.

### TOTP Replay Attack Prevention
Every time a TOTP code is successfully used, the current timestamp is written to `totp_last_used_at`. On the next verification attempt, if `totp_last_used_at` is within the current 30-second window, the code is rejected as a replay. This prevents the same code from being used twice in the same window.

### TOTP Secret Encryption
TOTP secrets are never stored in plaintext. They are encrypted with AES-256-GCM using the `TOTP_KEY` environment variable (a 32-byte hex key). The encrypted blob is stored in `accounts.totp_secret`.

### Step-Gated Tokens
Temp tokens carry a `step` field (`"pending_2fa"` or `"pending_password_change"`). The RBAC middleware rejects any temp token trying to reach a protected resource. Each step handler also validates that the correct step is present — you cannot call `verify-2fa` with a `pending_password_change` token or vice versa.

### Email Enumeration Prevention
`POST /auth/login` returns `401` with the message `"Invalid credentials"` for both wrong email and wrong password. It never reveals whether the email exists.

### Request Body Limit
Auth routes are wrapped with a 4 KB request body limit to prevent large-payload attacks.

### Password Strength Validation
`change-initial-password` enforces: ≥ 8 characters, at least 1 uppercase, 1 lowercase, 1 number.

---

## 9. Email / OTP System

### Terminology

- **OTP** (One-Time Password) — the 6-digit code sent by email when an account is created. Used as the initial login password. Must be changed on first login.
- **TOTP** (Time-based One-Time Password) — the 6-digit code generated every 30 seconds by an authenticator app. Used for 2FA on every login.

These are two different things. OTP is email-based and only used once. TOTP is app-based and used every login.

### Email Sending

Email is sent via SMTP using the `lettre` library. The function `send_otp_email` in `src/services/email.rs` handles this.

**Port-based transport selection:**
- Port `25` or `1025` → plain connection (no TLS) — used by Mailpit in tests
- Any other port (e.g., `587`) → STARTTLS with credentials — used by Gmail in production

### Production (Gmail) Setup

In `.env`:
```
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=youraddress@gmail.com
SMTP_PASS="your app password here"
SMTP_FROM=youraddress@gmail.com
```

Use a **Gmail App Password** (not your regular password). Create one at: Google Account → Security → 2-Step Verification → App passwords.

### Test (Mailpit) Setup

In `.env.test`:
```
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USER=test
SMTP_PASS=test
SMTP_FROM=admin@test.local
MAILPIT_URL=http://localhost:8025
```

Mailpit catches all outgoing email and exposes a web UI at `http://localhost:8025` and a JSON API for test assertions.

---

## 10. Environment Configuration

Full list of environment variables:

| Variable | Required | Default | Description |
|---|---|---|---|
| `DATABASE_URL` | Yes | — | PostgreSQL connection string |
| `JWT_SECRET` | Yes | — | Secret for signing JWTs |
| `JWT_EXPIRY_HOURS` | No | `24` | How long full tokens last |
| `TOTP_KEY` | Yes | — | 32-byte hex key for AES-256-GCM TOTP encryption |
| `SMTP_HOST` | Yes | — | SMTP server hostname |
| `SMTP_PORT` | No | `587` | SMTP port (1025 for Mailpit, 587 for Gmail) |
| `SMTP_USER` | Yes | — | SMTP username / email |
| `SMTP_PASS` | Yes | — | SMTP password or app password |
| `SMTP_FROM` | Yes | — | From address on sent emails |
| `MAILPIT_URL` | Test only | `http://localhost:8025` | Mailpit API URL (tests read this) |

---

## 11. Test Infrastructure

Tests are integration tests — they spin up a real Axum HTTP server on a random port, talk to a real PostgreSQL database, and send real SMTP emails (caught by Mailpit in tests).

### Key Files

| File | Purpose |
|---|---|
| `tests/common/mod.rs` | Shared test helpers: `spawn_app`, `create_test_account`, `login_as`, `generate_valid_totp` |
| `tests/create_account_test.rs` | Integration tests for account creation and management |
| `.env.test` | Environment for tests (loads automatically via `dotenvy`) |
| `db/docker-compose.test.yml` | Docker Compose for the test PostgreSQL + Mailpit |

### Helper Functions

**`spawn_app()`**
Starts the full Axum app on a random port. Connects to the test DB. Runs all migrations (idempotent). Returns a `TestApp` struct with `base_url`, `db`, `config`, and `mailpit_url`.

**`create_test_account(db, config, role, totp_enabled, is_active)`**
Inserts a test account directly into the DB. Generates a unique email per call. Uses bcrypt cost 4 (fast for tests). If `totp_enabled = true`, generates and encrypts a TOTP secret and returns the plaintext so tests can generate valid codes. Returns `(account_model, Option<plaintext_secret>)`.

**`login_as(base_url, email, plaintext_secret)`**
Full login flow shortcut: does `POST /auth/login` + `POST /auth/verify-2fa` and returns the `full_token`. Used in tests that don't want to repeat the 2-step login manually.

**`generate_valid_totp(plaintext_secret)`**
Generates the current valid TOTP code from a base32 secret. Uses SHA1, 6 digits, 30-second step — matching the production TOTP configuration exactly.

### Test Coverage

| Test | What it verifies |
|---|---|
| `test_create_super_moderator_sends_email` | Root creates a super mod → email is sent to Mailpit containing "OTP code" |
| `test_root_can_create_standard_moderator` | Root can POST /moderators/standard → 200 |
| `test_super_can_create_standard_moderator` | Super mod can POST /moderators/standard → 200 |
| `test_super_cannot_create_super_moderator` | Super mod cannot POST /moderators/super → 403 |
| `test_root_can_suspend_moderator` | Root can PATCH /accounts/{id}/status → 200 |
| `test_super_cannot_suspend_moderator` | Super mod cannot PATCH /accounts/{id}/status → 403 |
| `test_root_can_suspend_super_moderator` | Root can PATCH /moderators/super/{id}/status → 200 |
| `test_create_moderator_duplicate_email_returns_400` | Creating a second account with the same email → 400 |

---

## 12. Running Tests

### Prerequisites

1. Start the test Docker containers (PostgreSQL + Mailpit):
   ```bash
   docker compose -f db/docker-compose.test.yml up -d

   docker run -d -p 1025:1025 -p 8025:8025 axllent/mailpit
   ```

2. Make sure `.env.test` exists with the correct values (see section 10).

### Run all tests

```bash
cargo test
```

### Run a single test

```bash
cargo test test_root_can_create_standard_moderator
```

### Run a test file

```bash
cargo test --test create_account_test
```

### View emails caught by Mailpit

Open `http://localhost:8025` in a browser, or hit the API:
```bash
curl http://localhost:8025/api/v1/messages
```

### Common issues

**All tests fail with `PoolTimedOut`** — the test database container is not running. Run:
```bash
docker compose -f db/docker-compose.test.yml up -d
```

**Linker error LNK1104 (Windows)** — a previous test .exe is still locked. Kill it in Task Manager or close other terminal windows that ran tests.

**Test gets 403 unexpectedly** — the account role string doesn't match what RBAC expects. Valid roles are: `root_moderator`, `super_moderator`, `moderator`.

---

## 13. Architecture Notes

### Why verify-2fa and confirm-2fa are the same endpoint

Originally there were two separate TOTP endpoints:
- `confirm-2fa` — used during first-time TOTP setup (sets `totp_enabled = true`)
- `verify-2fa` — used on every subsequent login (does not set `totp_enabled`)

They were merged into a single `POST /auth/verify-2fa` because the only difference was one `if` statement: `if !totp_enabled { active.totp_enabled = Set(true); }`. Both endpoints accepted the same input, performed the same TOTP validation, and returned the same output. Having two routes was unnecessary complexity.

### Why the safe deletion is two steps

Deleting an account is irreversible. The two-step pattern (initiate → confirm) forces the caller to consciously confirm the deletion with a UUID that was just returned. This prevents accidental deletions from a single misfire POST request. The `confirmation_id` is stored in the account's `metadata` JSON column between the two calls.

### TOTP secret storage

TOTP secrets are AES-256-GCM encrypted before writing to the DB. The encryption key (`TOTP_KEY`) never touches the DB. Even if the `accounts` table were fully dumped, the TOTP secrets would be useless without the key. The nonce is prepended to the ciphertext in the stored blob.
