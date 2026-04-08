# Notes — How It Works

Notes are free-text entries attached to a **project**. Any moderator with access to that project can create, read, update, and delete notes.

---

## Database

```sql
CREATE TABLE "notes" (
    "id"         UUID        NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
    "project_id" UUID        NOT NULL REFERENCES "projects" ("id") ON DELETE CASCADE,
    "content"    TEXT        NOT NULL,
    "created_by" UUID        NOT NULL REFERENCES "accounts" ("id") ON DELETE NO ACTION,
    "created_at" TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    "updated_at" TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX "notes_project_id_idx" ON "notes" ("project_id");
```

- `project_id` → **CASCADE DELETE**: deleting a project wipes all its notes automatically.
- `created_by` → **NO ACTION**: deleting an account does not delete the notes they wrote.
- An index on `project_id` makes listing notes for a project fast.

---

## Access Control (`RequireProjectAccess`)

Every notes endpoint is guarded by the `RequireProjectAccess` middleware. Here is what it checks:

1. The request must have a valid JWT.
2. The caller's role must be one of: `root_moderator`, `super_moderator`, or `moderator`.
3. If the role is `root_moderator` or `super_moderator` → **pass through immediately**.
4. If the role is `moderator` → the middleware checks the `moderators` table to verify that this user is explicitly **assigned** to the project in the URL path. If not assigned → `403 Forbidden`.

In short: super-level moderators can touch any project's notes; standard moderators can only touch notes on projects they are assigned to.

---

## Endpoints

### Create a note
```
POST /projects/{project_id}/notes/create
Authorization: Bearer <token>
Content-Type: application/json

{ "content": "Your note text here" }
```
- `content` must be non-empty (validated before hitting the handler).
- Returns `201 Created` with the full note object.
- The `created_by` field is set automatically from the JWT — the client does not send it.

### Get all notes for a project
```
GET /projects/{project_id}/notes/get
Authorization: Bearer <token>
```
- Returns `200 OK` with an array of note objects.
- Returns every note that belongs to that `project_id`.

### Update a note
```
PUT /projects/{project_id}/notes/{note_id}/update
Authorization: Bearer <token>
Content-Type: application/json

{ "content": "Updated text" }
```
- Looks up the note by both `note_id` AND `project_id` (prevents touching notes across projects).
- Returns `200 OK` with the updated note.
- Returns `404 Not Found` if the note does not exist under that project.
- `updated_at` is bumped automatically.

### Delete a note
```
DELETE /projects/{project_id}/notes/{note_id}/delete
Authorization: Bearer <token>
```
- Deletes the note matching both `note_id` and `project_id`.
- Returns `200 OK` (even if the note did not exist — no error on missing).

---

## DTO (what the API sends and receives)

```json
{
  "id":         "uuid (omitted on create request)",
  "project_id": "uuid (omitted on create request)",
  "content":    "string — required, min length 1",
  "created_by": "uuid (omitted on create request)",
  "created_at": "timestamp (omitted on create request)",
  "updated_at": "timestamp (omitted on create request)"
}
```

Fields marked *omitted on create request* are set by the server. The client only needs to send `content`.

---

## Data Flow (create as example)

```
Client
  └─ POST /projects/{project_id}/notes/create  { "content": "..." }
       │
       ├─ JWT extracted → role checked (must be moderator or above)
       ├─ If standard moderator → DB check: is this user assigned to project_id?
       │    └─ No  → 403 Forbidden
       │    └─ Yes → continue
       │
       ├─ Payload validated (content non-empty)
       │    └─ Fails → 422 Unprocessable Entity
       │
       └─ INSERT into notes (project_id, content, created_by = jwt.sub, updated_at = now())
            └─ 201 Created + NoteDto
```

---

## Cascade Behavior

When a project is deleted, all of its notes are automatically removed by the database (`ON DELETE CASCADE`). The notes endpoints for that project will subsequently return an empty list (the `RequireProjectAccess` guard still passes for super moderators, but there are no rows to return).
