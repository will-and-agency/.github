# Mobile — Projects & Tasks (feature/49)

These are the read-only endpoints built for the mobile app (audience users / participants). They let a logged-in participant see the projects they are enrolled in and the tasks inside each project.

---

## Who can use these routes

Only accounts with the role `audience_user` (participants). Moderator tokens are explicitly rejected with `403`.

---

## Access Control

### `RequireAudienceUser`
Used on `/me/projects/get`.

1. Extract and validate the JWT.
2. Check `claims.role == "audience_user"` — any other role → `403 Forbidden`.
3. Pass through. No DB check needed; the query itself filters by the caller's ID.

### `RequireParticipantAccess`
Used on `/me/projects/{project_id}/tasks/get`.

1. Extract and validate the JWT.
2. Check `claims.role == "audience_user"` — any other role → `403 Forbidden`.
3. Query the `participants` table: does a row exist for `(user_id = jwt.sub, project_id = path param)`?
   - No row → `403 Forbidden`.
   - Row found → pass through.

The key difference: `RequireAudienceUser` only checks the role; `RequireParticipantAccess` also verifies the user is actually enrolled in the specific project in the URL.

---

## Endpoints

### Get my projects
```
GET /me/projects/get
Authorization: Bearer <audience_user_token>
```
Returns all projects the authenticated participant is enrolled in (joined through the `participants` table).

- `200 OK` — array of `ProjectDto` (may be empty if not enrolled anywhere).
- `401 Unauthorized` — no or invalid token.
- `403 Forbidden` — valid token but not an `audience_user`.

### Get tasks for a project
```
GET /me/projects/{project_id}/tasks/get
Authorization: Bearer <audience_user_token>
```
Returns all tasks belonging to `project_id`, **ordered by `sort_order` ascending**.

- `200 OK` — array of `TaskDto` (may be empty if the project has no tasks).
- `401 Unauthorized` — no or invalid token.
- `403 Forbidden` — not an `audience_user`, or `audience_user` not enrolled in this project.

---

## DTOs

### ProjectDto (response)
```json
{
  "id":          "uuid",
  "company":     "string",
  "name":        "string",
  "description": "string | null",
  "status":      "draft | active | completed | archived",
  "settings":    "object | null",
  "starts_at":   "timestamp | null",
  "ends_at":     "timestamp"
}
```

### TaskDto (response)
```json
{
  "id":          "uuid",
  "project_id":  "uuid",
  "title":       "string",
  "description": "string",
  "sort_order":  123,
  "config":      {}
}
```

`config` is a freeform JSON blob — its shape is defined per task by the admin who created it.

Tasks are always returned sorted by `sort_order` ascending. The mobile app can render them in the order they arrive without sorting client-side.

---

## Data Flow

### GET /me/projects/get
```
Participant
  └─ GET /me/projects/get
       │
       ├─ JWT extracted → role must be "audience_user" → else 403
       │
       └─ SELECT projects.*
            INNER JOIN participants ON participants.project_id = projects.id
            WHERE participants.user_id = jwt.sub
              └─ 200 OK + [ProjectDto, ...]
```

### GET /me/projects/{project_id}/tasks/get
```
Participant
  └─ GET /me/projects/{project_id}/tasks/get
       │
       ├─ JWT extracted → role must be "audience_user" → else 403
       ├─ DB check: participants WHERE user_id = jwt.sub AND project_id = :project_id
       │    └─ Not found → 403 Forbidden
       │
       └─ SELECT * FROM tasks
            WHERE project_id = :project_id
            ORDER BY sort_order ASC
              └─ 200 OK + [TaskDto, ...]
```

---

## Important Rules

- **Participants only.** These routes are strictly separated from the moderator/admin surface. A moderator cannot call `/me/...` routes even if they are also enrolled as a participant — the role check blocks them first.
- **Enrollment check is per-project.** Being enrolled in project A does not give access to project B's tasks.
- **Tasks are not filtered by participant.** All tasks in the project are returned — the filtering is at the project enrollment level, not the task level.
- **Read-only.** There are no create/update/delete operations on this route group. Participants consume data; moderators manage it through the separate `/projects/...` routes.
