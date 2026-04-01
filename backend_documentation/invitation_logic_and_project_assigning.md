# Moderator Project Assignment & Participant Invites

This document covers two core workflows:
1. How moderators are assigned to projects and what that unlocks
2. How participants are invited to a project via CSV upload and email

---

## Part 1 — Moderator Project Assignment

### Overview

A moderator account exists independently of any project. To give a moderator access to a specific project — and the ability to upload participants or send invites for it — they must be explicitly assigned to that project. This is done by a super moderator or root moderator.

### Database tables involved

**`accounts`** — stores every person who can log in, regardless of role.

**`moderators`** — the assignment table. A row here means "this user is a moderator on this project."

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Auto-generated |
| `project_id` | UUID | FK → projects |
| `user_id` | UUID | FK → accounts |
| `created_at` | TIMESTAMP | When the assignment was made |

The combination of `(project_id, user_id)` is unique — a moderator can only be assigned to the same project once.

### How it works

When a super/root moderator calls the assign endpoint, the backend:

1. Verifies the target account exists and has role `"moderator"`.
2. For each project ID in the request, verifies the project exists.
3. Checks if the assignment already exists — if it does, silently skips it (idempotent).
4. If it doesn't exist, inserts a row into `moderators`.
5. All inserts happen inside a single **database transaction** — if any project ID is invalid, the whole batch is rolled back.

### Endpoints

---

#### Assign projects to a moderator

```
POST /moderators/standard/{id}/assign-project
```

**Auth:** Super moderator or root moderator JWT required.

**Path param:** `{id}` — the UUID of the moderator account to assign projects to.

**Request body:**
```json
{
  "project_ids": ["uuid-1", "uuid-2"]
}
```

**Response:** `200 OK` — no body. Silently skips already-assigned projects.

**Errors:**
- `404` — moderator account not found, or one of the project IDs doesn't exist
- `400` — the target account is not a moderator
- `403` — caller is not a super/root moderator

---

#### List projects assigned to a moderator

```
GET /moderators/standard/{id}/projects
```

**Auth:** Super moderator or root moderator JWT required.

**Response:** `200 OK` — array of project objects.
```json
[
  {
    "id": "uuid",
    "name": "Project Name",
    "company": "Company",
    "status": "active",
    "ends_at": "2026-05-15T00:00:00Z"
  }
]
```

---

#### Remove a project assignment

```
DELETE /moderators/standard/{id}/deassign-project/{project_id}
```

**Auth:** Super moderator or root moderator JWT required.

**Response:** `200 OK`

**Errors:**
- `404` — assignment not found (moderator was not assigned to this project)

---

### Why assignment matters

The `RequireProjectAccess` middleware used on participant routes checks the `moderators` table to confirm the authenticated user is assigned to the project in the URL. Without an assignment row, the request returns `403` — even if the user is a valid moderator. This is what scopes each moderator's access to only their projects.

---

## Part 2 — Participant Invites

### Overview

Inviting participants is a two-step process:

1. **Upload CSV** — parse the file, create one account and one participant record per email row.
2. **Send Invites** — send the HTML email with login credentials to every participant who hasn't received one yet.

The two steps are intentionally separate so the frontend can show a preview of who will be created before committing to sending emails.

### Database tables involved

**`accounts`** — one row per participant, same table as moderators but with role `"audience_user"`.

| Column | Participant value |
|---|---|
| `email` | From CSV |
| `password_hash` | bcrypt hash of the generated OTP |
| `first_name` | `NULL` — not stored for participants |
| `last_name` | `NULL` — not stored for participants |
| `role` | `"audience_user"` |
| `is_active` | `true` |
| `metadata` | `{ "must_change_password": true }` |

**`participants`** — links an account to a project and tracks invitation state.

| Column | Value at creation |
|---|---|
| `project_id` | From the route path |
| `user_id` | The `accounts.id` just created |
| `status` | `"pending"` |
| `invited_by` | The authenticated moderator's account ID (from JWT) |
| `invited_at` | Current UTC time |
| `accepted_at` | `NULL` until the participant accepts |
| `metadata` | CSV columns + temporary credentials (see below) |

#### What lives in `participants.metadata`

Immediately after CSV upload:
```json
{
  "city": "Oslo",
  "age": "30",
  "temp_password": "048293",
  "credentials_sent": false
}
```

After send invites:
```json
{
  "city": "Oslo",
  "age": "30",
  "credentials_sent": true
}
```

All columns from the CSV except `email` are stored here permanently. `temp_password` is removed the moment the invite email is sent — it only exists in the DB during the window between upload and send.

---

### Step 1 — Upload CSV

#### Endpoint

```
POST /projects/{project_id}/participants/upload-csv
```

**Auth:** Valid JWT. The authenticated user must be assigned to the project (`RequireProjectAccess`).

**Request:** `multipart/form-data` with a field named `file` containing the CSV.

**CSV requirements:**
- Must have a header row.
- Must contain an `email` column (case-insensitive — `Email`, `EMAIL`, `email` all work).
- All other columns are optional and stored as metadata on the participant record.

**Example CSV:**
```csv
email,age,city
alice@example.com,34,Oslo
bob@example.com,28,Bergen
```

#### What happens

For each row in the CSV:

1. Email is trimmed and lowercased.
2. Empty email values are silently skipped.
3. The `accounts` table is checked for that email — if it already exists, the email is added to `skipped_emails` and the row is skipped.
4. All non-email columns are collected into a JSON object for participant metadata.
5. A 6-digit numeric OTP is generated (e.g. `"048293"`) using `rand`.
6. `create_account` is called — hashes the OTP with bcrypt and inserts the account with role `"audience_user"`, no first/last name, and `must_change_password: true`.
7. `temp_password` and `credentials_sent: false` are added to the metadata map.
8. A `participants` row is inserted linking the account to the project with `status: "pending"`.

#### Response

```json
{
  "created": 2,
  "skipped": 0,
  "results": [
    {
      "email": "alice@example.com",
      "account_id": "uuid-...",
      "participant_id": "uuid-..."
    }
  ],
  "skipped_emails": []
}
```

| Field | Description |
|---|---|
| `created` | Number of accounts + participant records successfully created |
| `skipped` | Number of emails that already had accounts |
| `results` | Per-row detail of what was created |
| `skipped_emails` | List of emails that were skipped |

---

### Step 2 — Send Invites

#### Endpoint

```
POST /projects/{project_id}/participants/send-invites
```

**Auth:** Valid JWT. The authenticated user must be assigned to the project.

**Request body:**
```json
{
  "reward_amount": "500 NOK GoGift gift card",
  "reward_deadline": "2 weeks after the survey deadline",
  "manager_email": "manager@willagency.no",
  "language": "en"
}
```

| Field | Required | Description |
|---|---|---|
| `reward_amount` | Yes | Inserted into the email as `{{REWARD_AMOUNT}}` |
| `reward_deadline` | Yes | Inserted as `{{REWARD_DEADLINE}}` |
| `manager_email` | No | Contact email shown in the template. Defaults to the authenticated moderator's own email |
| `language` | Yes | `"en"` for English, `"dk"` for Danish |

#### What happens

1. The project is loaded from the DB — used for `{{PROJECT_NAME}}` and `{{PROJECT_DEADLINE}}` (formatted from `projects.ends_at`).
2. Manager email is resolved — uses the request body value if provided, otherwise looks up the authenticated user's email from `accounts`.
3. Task count is queried from the `tasks` table for `{{NUMBER_OF_TASKS}}`.
4. All participant records for the project are loaded.
5. For each participant:
   - If `metadata.credentials_sent` is `true` → skip, count as `already_sent`.
   - If `metadata.temp_password` is missing → skip, count as `already_sent`.
   - Fetches the participant's email from `accounts` via `user_id`.
   - Builds the HTML email by replacing all `{{PLACEHOLDER}}` tokens in the template.
   - Sends the email via SMTP.
   - Updates the participant's metadata: removes `temp_password`, sets `credentials_sent: true`.
6. Returns the sent/already_sent counts.

This endpoint is **idempotent** — calling it multiple times will not re-send to participants who already received their invite.

#### Response

```json
{
  "sent": 2,
  "already_sent": 0
}
```

---

### Email template variables

Both templates (`email.html` for English, `email_dk.html` for Danish) use these placeholders:

| Placeholder | Value | Source |
|---|---|---|
| `{{PARTICIPANT_EMAIL}}` | `alice@example.com` | `accounts.email` looked up via `participants.user_id` |
| `{{TEMP_PASSWORD}}` | `048293` | `participants.metadata.temp_password` |
| `{{PROJECT_NAME}}` | `"The Great Heist"` | `projects.name` |
| `{{PROJECT_DEADLINE}}` | `"May 15, 2026"` | `projects.ends_at` formatted as `Month DD, YYYY` |
| `{{NUMBER_OF_TASKS}}` | `6` | Count of rows in `tasks` where `project_id` matches |
| `{{REWARD_AMOUNT}}` | `"500 NOK"` | Request body |
| `{{REWARD_DEADLINE}}` | `"2 weeks after..."` | Request body |
| `{{MANAGER_EMAIL}}` | `"mgr@agency.no"` | Request body, or authenticated user's email |
| `{{ORG_NAME}}` | `"PUBLIKUM"` | Hardcoded |
| `{{ORG_ADDRESS}}` | `"Will & Agency"` | Hardcoded |

---

### The OTP password

The temporary password sent to participants is a **6-digit numeric OTP** (e.g. `"048293"`), generated the same way as moderator accounts:

```
format!("{:06}", rand::thread_rng().gen_range(0..1_000_000))
```

It is:
- Stored as a bcrypt hash in `accounts.password_hash` — this is their actual login password.
- Stored in plaintext temporarily in `participants.metadata.temp_password` — so it can be included in the email.
- Deleted from metadata the moment the email is successfully sent.
- Used by the participant to log in for the first time. Because `must_change_password: true` is set on their account, they will be prompted to set a new password on first login.

---

### Security notes

- `temp_password` only exists in the DB during the window between CSV upload and send invites. After sending it is deleted.
- The plaintext password is never returned in any API response.
- Participants have no first or last name stored — only email and role.
- Project access is scoped per moderator via the `moderators` table — a moderator can only upload/invite for projects they are explicitly assigned to.

---

## Full flow summary

```
Super moderator
  └── Creates project
  └── Creates standard moderator account
  └── Assigns moderator to project  →  row in moderators table

Moderator (assigned to project)
  └── Uploads CSV
        └── For each email row:
              ├── Creates account (audience_user, no name, OTP password)
              └── Creates participant record (status: pending, metadata: CSV cols + temp_password)
  └── Sends invites
        └── For each unsent participant:
              ├── Builds HTML email with project/reward/credential variables
              ├── Sends via SMTP
              └── Cleans temp_password from metadata, sets credentials_sent: true

Participant
  └── Receives email
  └── Downloads AppPublikum
  └── Logs in with email + OTP
  └── Changes password
  └── Completes survey tasks
```
