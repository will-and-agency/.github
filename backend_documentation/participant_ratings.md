# Participant Rating Feature

A moderator can rate each participant's answer on a 1–5 star scale. Ratings roll up into a global average stored on the participant's account and are exposed through the participant overview and detail APIs.

---

## Database

### Migration

**File:** `migration/src/m20260422_131112_add_answer_ratings.rs`

Two changes are applied in one migration:

**1. New table: `answer_ratings`**

```sql
CREATE TABLE answer_ratings (
    answer_id   UUID        PRIMARY KEY,
    rated_by    UUID        NOT NULL,
    rating      SMALLINT    NOT NULL,
    rated_at    TIMESTAMPTZ NOT NULL,
    FOREIGN KEY (answer_id) REFERENCES answers(id) ON DELETE CASCADE
);
```

- `answer_id` is the primary key — there is at most **one rating per answer**.
- `rated_by` is the moderator UUID who submitted the rating.
- `rating` stores values 1–5.
- Cascade delete: if the answer is deleted, the rating goes with it.

**2. New column on `accounts`**

```sql
ALTER TABLE accounts ADD COLUMN average_rating DECIMAL(3, 2);
```

- Nullable — `NULL` means the participant has never been rated.
- Global: reflects the average across **all projects**, not just one.
- Stored as `DECIMAL(3,2)` — e.g. `3.75`.

---

## Entities (SeaORM)

### `answer_ratings` entity

**File:** `src/entities/answer_ratings.rs`

```rust
pub struct Model {
    pub answer_id: Uuid,        // PK
    pub rated_by: Uuid,
    pub rating: i16,
    pub rated_at: DateTimeWithTimeZone,
}
```

Relation: `belongs_to answers::Entity` via `answer_id`.

### `accounts` entity

The existing `accounts::Model` gained:

```rust
pub average_rating: Option<Decimal>,
```

Uses `rust_decimal::Decimal` so precision is not lost in transit from the database.

---

## API Endpoints

Both endpoints share the same URL — one verb for each action.

### POST — Rate an answer

```
POST /api/projects/{project_id}/tasks/{task_id}/questions/{question_id}/answers/{answer_id}/rate
```

**Auth:** `RequireAdminProjectAccess` — moderator or super_moderator assigned to the project, or root_moderator.

**Request body:**

```json
{ "rating": 4 }
```

`rating` must be an integer between 1 and 5 (inclusive). Anything outside that range returns `400 Bad Request`.

**What it does:**

1. Validates the rating range.
2. Looks up the answer to find its owner (`participant_id`).
3. Deletes any existing `answer_ratings` row for this `answer_id` (the old rating), then inserts the new one — this is an **upsert by delete+insert**.
4. Runs a cross-project average recalculation:
   - Inner-joins `answer_ratings` with `answers` filtered to `answers.user_id = participant_id`.
   - Computes the mean of all ratings, rounded to 2 decimal places.
   - Updates `accounts.average_rating` for the participant.

**Response `200 OK`:**

```json
{
  "answer_id": "uuid",
  "rating": 4,
  "rated_by": "moderator-uuid",
  "rated_at": "2026-04-22T13:11:12+00:00"
}
```

---

### GET — Fetch existing rating

```
GET /api/projects/{project_id}/tasks/{task_id}/questions/{question_id}/answers/{answer_id}/rate
```

**Auth:** same as POST (`RequireAdminProjectAccess`).

**Response `200 OK`:**

Returns the saved rating if one exists, or `null` if the answer has never been rated.

```json
{
  "answer_id": "uuid",
  "rating": 4,
  "rated_by": "moderator-uuid",
  "rated_at": "2026-04-22T13:11:12+00:00"
}
```

or

```json
null
```

---

## Handler

**File:** `src/handlers/answer.rs`

Two handler functions, registered on the same route with different HTTP verbs:

| Function | Verb | Purpose |
|---|---|---|
| `rate_answer` | POST | Upsert rating + recalculate average |
| `get_answer_rating` | GET | Read current rating |

**Route registration** (`src/routes/answer.rs`):

```rust
.route(
    "/projects/{project_id}/tasks/{task_id}/questions/{question_id}/answers/{answer_id}/rate",
    post(answer::rate_answer).get(answer::get_answer_rating),
)
```

---

## Average Rating Calculation

The average is **global** — it reflects every answer rated across every project the participant has ever been in.

```
average = sum(all ratings for participant) / count(all ratings for participant)
```

Rounded to 2 decimal places before writing to `accounts.average_rating`.

Example: participant has answer A rated 2 in project 1 and answer B rated 4 in project 2 → average = `(2 + 4) / 2 = 3.00`.

---

## Participant Overview & Detail

`average_rating` is surfaced on two read endpoints.

### Overview list

```
GET /api/projects/{project_id}/participants/get
```

Each item in the `participants` array includes:

```json
{
  "participant_id": "...",
  "account_id": "...",
  "email": "...",
  "average_rating": 3.75
}
```

`average_rating` is `null` when the participant has no ratings yet.

### Participant detail

```
GET /api/projects/{project_id}/participants/{participant_id}/get
```

Same field:

```json
{
  "participant_id": "...",
  "average_rating": 3.75
}
```

Both are handled in `src/handlers/participants.rs`. The `Decimal` stored in the DB is converted to `f64` for the JSON response:

```rust
average_rating: account.average_rating.map(|d| d.to_string().parse::<f64>().unwrap_or(0.0))
```

DTOs are in `src/dto/participants.rs` (`ParticipantOverviewItem` and `ParticipantDetailResponse`).

---

## Frontend Web

### Service layer

**File:** `src/services/attachmentService.ts`

```ts
// Submit or update a rating
rateAnswer(projectId, taskId, questionId, answerId, rating)  // POST .../rate

// Fetch existing rating (returns number | null)
getAnswerRating(projectId, taskId, questionId, answerId)     // GET  .../rate
```

Both use the same URL path ending in `/rate`.

**File:** `src/services/participantService.ts`

`ParticipantOverviewItem` and `ParticipantDetail` both carry:

```ts
average_rating?: number | null
```

### UI (`ProjectDetail.tsx`)

When a moderator opens a participant's answer in the chat panel, a star rating strip appears below the answer body.

**Behaviour:**

- On open, `getAnswerRating` is called to fetch any existing rating. If one exists it is pre-filled.
- While no rating is confirmed the stars are interactive (hover/click selects a pending value).
- Pressing **Confirm** calls `rateAnswer`, locks the strip, and shows the confirmed score with an **Edit** button.
- Pressing **Edit** re-enters the interactive state with the previously saved value pre-selected.
- Pressing **Cancel** (only available in edit mode) restores the last confirmed value without making an API call.
- On chat close all rating state is reset.

**Participant overview table** — "Rating" column shows filled/empty stars and the decimal value (e.g. `★★★☆☆ 3.5`) or `—` when unrated.

**Participant profile modal** — an "Avg. Rating" info cell shows the same stars + decimal display.

---

## Tests

**File:** `tests/participants/rating_tests.rs`

10 integration tests (RATE-01 through RATE-10) cover the full feature end-to-end using a real database.

| Test | What it checks |
|---|---|
| RATE-01 | POST rating returns 200 with correct fields |
| RATE-02 | Rating outside 1–5 returns 400 |
| RATE-03 | Unauthenticated request returns 401 |
| RATE-04 | GET returns the saved rating value |
| RATE-05 | GET returns null when never rated |
| RATE-06 | Second rating on same answer upserts (last value wins) |
| RATE-07 | `average_rating` on the account row is updated after rating |
| RATE-08 | `average_rating` is global — a second project's rating changes the mean |
| RATE-09 | `average_rating` appears in the participant overview list endpoint |
| RATE-10 | `average_rating` appears in the participant detail endpoint |

Each test spawns a fresh in-process Axum server with its own DB connection and runs the full HTTP flow (create project → assign moderator → create task → create question → enroll participant → submit answer → rate).
