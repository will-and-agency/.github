# Backend — Notifications & Automatic Project Statuses

## Overview

This document covers two features added to the Rust/Axum backend:

1. **Notifications system** — real-time WebSocket notifications + REST endpoints for both admins and participants
2. **Automatic project status lifecycle** — projects transition through `draft → active → archived` automatically based on their dates, with no manual status changes allowed

---

## Part 1 — Notifications

### What was built

The notifications system allows admins to see when participants send chat messages they haven't read yet, and allows participants to see when an admin replies to them. There is also support for mobile push notifications via Expo.

There are two completely separate notification flows:

- **Admin notifications** — an admin (moderator/super_moderator/root_moderator) sees a badge/list of conversations with unread participant messages
- **Participant notifications** — a participant sees a badge/list of questions where an admin replied

---

### Database tables

Three new tables support this system:

#### `push_tokens`
Stores Expo push tokens for mobile participants so the server can send push notifications to their device.

| Column | Type | Description |
|---|---|---|
| `id` | UUID (PK) | Auto-generated |
| `account_id` | UUID (FK → accounts) | Which user owns this token |
| `token` | TEXT | The Expo push token string |
| `created_at` | TIMESTAMPTZ | When the token was registered |

Cascade delete: if the account is deleted, their push tokens are automatically removed.

#### `admin_chat_reads`
Tracks which admin has read which conversation room (identified by `question_id + participant_id`), and whether they have dismissed it from the notification tab.

| Column | Type | Description |
|---|---|---|
| `account_id` | UUID (PK) | The admin's account |
| `question_id` | UUID (PK) | Which question the chat is about |
| `participant_id` | UUID (PK) | Which participant the chat is with |
| `last_read_at` | TIMESTAMPTZ | Last time the admin read this room (clears badge count) |
| `dismissed_at` | TIMESTAMPTZ (nullable) | If set, the notification is hidden until a newer message arrives |

The composite primary key `(account_id, question_id, participant_id)` ensures one record per admin per room.

#### `participant_chat_dismissals`
Tracks when a participant dismisses a notification for a specific question chat.

| Column | Type | Description |
|---|---|---|
| `account_id` | UUID (PK) | The participant's account |
| `question_id` | UUID (PK) | Which question they dismissed |
| `dismissed_at` | TIMESTAMPTZ | When they dismissed it |

---

### API endpoints

All notification endpoints live under the `notifcations_router` (note: typo in code, keep consistent) and are mounted at `/api`.

#### Push token management (participants only)

**`POST /api/push-tokens`**
Registers a new Expo push token for the authenticated participant. If the same token already exists for this user it is deleted first then re-inserted (deduplication).

**`DELETE /api/push-tokens`**
Removes a push token. Called when a participant logs out or uninstalls the app.

Both require `role = "audience_user"`. Admins cannot register push tokens.

---

#### Admin notifications

**`GET /api/notifications`**

Returns a list of conversation rooms that have unread or undismissed participant messages. Only rooms where the **participant was the last sender** (i.e. the participant sent a message the admin hasn't dismissed) appear here.

Logic:
1. Fetch all `admin_chat_reads` records for the current admin
2. Build two maps: `read_map` (last_read_at per room) and `dismissed_map` (dismissed_at per room)
3. Fetch all messages. For moderators, only fetch messages from their assigned projects
4. Keep only messages where `sender_id == participant_id` (participant sent it) AND either never dismissed, or sent after `dismissed_at`
5. Group by `(project_id, participant_id, question_id)` and compute `unread_count` as messages after `last_read_at`
6. Return sorted by latest message descending

Response shape per item:
```json
{
  "project_id": "uuid",
  "project_name": "string",
  "participant_id": "uuid",
  "participant_email": "string",
  "question_id": "uuid",
  "unread_count": 3,
  "latest_message": "string",
  "latest_at": "2026-04-22T10:00:00Z"
}
```

**`POST /api/notifications/mark-read`**

Marks a specific room as read (resets the unread badge count). Does NOT dismiss the notification from the tab — it just updates `last_read_at`.

Body:
```json
{ "question_id": "uuid", "participant_id": "uuid" }
```

Upserts the `admin_chat_reads` record.

**`POST /api/notifications/dismiss`**

Fully dismisses a notification room. Sets `dismissed_at` to now. The room will no longer appear in the list unless the participant sends another message after this timestamp.

Body:
```json
{ "question_id": "uuid", "participant_id": "uuid" }
```

---

#### Participant notifications

**`GET /api/participant/notifications`**

Returns a list of question chats where an admin replied and the participant hasn't dismissed it yet. Only shows messages where `sender_id != participant_id` (admin sent it) AND sent after the participant's last `dismissed_at` for that question.

Response shape per item:
```json
{
  "project_id": "uuid",
  "project_name": "string",
  "question_id": "uuid",
  "question_description": "string",
  "unread_count": 1,
  "latest_message": "string",
  "latest_at": "2026-04-22T10:00:00Z"
}
```

The `unread_count` for participants is simply the count of messages where `read_at IS NULL`.

**`POST /api/participant/notifications/dismiss`**

Dismisses a specific question's notification for the authenticated participant. Deletes any existing dismissal record then inserts a fresh one with `dismissed_at = now`.

Body:
```json
{ "question_id": "uuid" }
```

---

#### Real-time WebSocket

**`GET /api/ws/notifications?token=<jwt>`**

Upgrades an HTTP connection to a WebSocket. The token must be passed as a **query parameter** (not a header) because browsers cannot set custom headers on WebSocket connections.

Authentication: the token is verified before the upgrade. If invalid or a step-token (2FA partial token), the connection is rejected with 401.

How it works:
- Each connected user gets a `broadcast::Sender<String>` stored in `AppState.notification_channels` (a `DashMap<Uuid, broadcast::Sender<String>>`)
- When any part of the backend wants to push a real-time notification to a user, it looks up their channel in `notification_channels` and sends a JSON string
- The WebSocket handler subscribes to that channel and forwards every message to the client
- If the client disconnects, the loop exits and the subscription is dropped

The channel has a buffer of 64 messages. Lagged messages (buffer overflow) are silently skipped (`RecvError::Lagged`).

**Important:** The WebSocket currently only delivers push signals — the client is expected to re-fetch `/api/notifications` on receipt. The WS message format is whatever JSON string is sent into the broadcast channel by the sender.

---

### Where notifications are triggered

Notifications are sent from the message handler when a new chat message is saved. The backend looks up the recipient's channel in `notification_channels` and broadcasts a JSON payload. If the recipient has no active WebSocket connection (no entry in the map), the send is a no-op.

---

## Part 2 — Automatic Project Statuses

### What changed and why

Previously, project status (`active`, `draft`, `completed`, `archived`) was a plain string stored in the database and could be manually set by admins on create or update. This caused problems:

- Seeded test projects had past `ends_at` dates. Any update attempt would be rejected by the date validator, blocking even cover image uploads (which were chained after the update call)
- Manual status created inconsistencies — a project could be "active" even after its end date passed
- The `completed` status had no clear meaning distinct from `archived`

The solution: **status is now computed from dates at read time, never stored manually**.

---

### The status lifecycle

```
Created → draft
starts_at reached → active   (or immediately active if no starts_at)
ends_at reached → archived
```

| Condition | Status |
|---|---|
| `ends_at < now` | `archived` |
| `starts_at > now` (and `ends_at` is future) | `draft` |
| anything else | `active` |

Archived takes absolute priority — if `ends_at` is past, the project is archived regardless of `starts_at`.

---

### Code changes

#### `src/services/project.rs`

- Removed `Status::Completed` from the enum
- Updated `to_string()` and `From<String>` to only handle `active`, `draft`, `archived`
- Added `pub fn compute_status(starts_at: Option<DateTimeWithTimeZone>, ends_at: DateTimeWithTimeZone) -> Status`

```rust
pub fn compute_status(
    starts_at: Option<DateTimeWithTimeZone>,
    ends_at: DateTimeWithTimeZone,
) -> Status {
    let now = Utc::now();
    if ends_at < now {
        return Status::Archived;
    }
    match starts_at {
        Some(start) if start > now => Status::Draft,
        _ => Status::Active,
    }
}
```

#### `src/dto/projects.rs`

- Removed `validate_status` function entirely (no manual status validation needed)
- Updated `From<projects::Model> for ProjectDto` to call `compute_status` instead of copying `model.status`:

```rust
impl From<projects::Model> for ProjectDto {
    fn from(project: projects::Model) -> Self {
        let status = compute_status(project.starts_at, project.ends_at).to_string();
        ProjectDto {
            status: Some(status),
            // ... rest of fields
        }
    }
}
```

- `validate_project_dates` kept unchanged: rejects `ends_at < now` and `ends_at <= starts_at`

#### `src/handlers/project.rs`

**`create_project`**: initial DB status set to `Draft` (was `Active`):
```rust
status: Set(Status::Draft.to_string()),
```
This is just the initial DB value. The response already returns the correct computed status because the insert result is converted through the `From` impl.

**`update_project`**: the block that updated status from the payload was removed:
```rust
// REMOVED — status is computed, not stored:
// if let Some(status) = payload.status {
//     project.status = Set(status);
// }
```

#### What this means in practice

- The `status` column in the `projects` table still exists and is still written on create (`draft`), but it is **never read for display** — every GET response computes it fresh
- No migration is needed — the column stays, it just becomes a vestigial initial value
- Passing `"status"` in an update payload is silently ignored — it does not cause an error, it just has no effect
- The frontend does not need to know about this — it just reads whatever status comes back in the response

---

### Tests updated

#### `tests/project/project_unit_test.rs`

- Removed all `validate_project_status` tests (function no longer exists)
- Removed `accepts_completed_status` test (`Status::Completed` no longer exists)
- Added 5 `compute_status` unit tests covering all branches:
  - archived when `ends_at` is past
  - draft when `starts_at` is future
  - active when no `starts_at` and `ends_at` is future
  - active when `starts_at` is past and `ends_at` is future
  - archived takes priority even if `starts_at` is also future (edge case)
- Fixed `from_model_maps_all_fields`: `dto.status` is now compared to `compute_status(...)` not `model.status`

#### `tests/project/project_tests.rs`

- `test_update_project_success`: removed `"status": "archived"` from the update payload (it is ignored) and corrected the assertion to `"active"` (the project has a future `ends_at` with no `starts_at`)
- `test_create_project_success_returns_201`: assertion `"active"` is still correct because `compute_status(None, future_date)` returns `Active`
