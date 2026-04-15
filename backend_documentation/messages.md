# Messages & Real-Time Chat Feature

Everything about how the chat system works: database, backend APIs, WebSocket, and both frontends.

---

## Table of Contents

1. [Overview](#overview)
2. [Database Schema](#database-schema)
3. [Backend — DTOs](#backend--dtos)
4. [Backend — Routes](#backend--routes)
5. [Backend — WebSocket Handler](#backend--websocket-handler)
6. [Backend — REST Endpoints](#backend--rest-endpoints)
7. [Backend — State & Broadcast Rooms](#backend--state--broadcast-rooms)
8. [Web Frontend](#web-frontend)
9. [Mobile Frontend](#mobile-frontend)
10. [Auth Flow for WebSocket](#auth-flow-for-websocket)
11. [Room Key — Critical Detail](#room-key--critical-detail)
12. [Message Flow Diagram](#message-flow-diagram)

---

## Overview

Each chat is scoped to a **room** identified by `(question_id, participant_id)`. A room is a private 1-on-1 conversation between one participant (audience_user) and the admin team about a specific question.

- Messages are persisted to the database.
- Real-time delivery uses tokio broadcast channels — all clients connected to the same room receive every new message immediately.
- The admin (web) connects to a participant's room using the participant's account UUID and their own admin JWT.
- The participant (mobile) connects to their own room using their own JWT.

---

## Database Schema

**Table: `messages`**

```sql
CREATE TABLE "messages" (
    "id"             UUID        NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
    "question_id"    UUID        NOT NULL REFERENCES "questions"  ("id") ON DELETE CASCADE,
    "participant_id" UUID        NOT NULL REFERENCES "accounts"   ("id") ON DELETE CASCADE,
    "project_id"     UUID        NOT NULL REFERENCES "projects"   ("id") ON DELETE CASCADE,
    "sender_id"      UUID        NOT NULL REFERENCES "accounts"   ("id") ON DELETE CASCADE,
    "content"        TEXT        NOT NULL,
    "created_at"     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    "read_at"        TIMESTAMPTZ
);
```

**Key columns:**

| Column | Description |
|---|---|
| `participant_id` | The participant (audience_user) this conversation belongs to. Always the participant's `accounts.id` (account UUID). |
| `sender_id` | Who actually sent this message — either the participant's or the admin's `accounts.id`. |
| `read_at` | NULL until the other party reads it. Used for unread counts. |

**Migration file:** `migration/src/m20260415_095401_create_messages.rs`

---

## Backend — DTOs

**File:** `src/dto/messages.rs`

```rust
// Sent by a client over WebSocket to post a new message
pub struct IncomingMessage {
    pub content: String,
}

// Broadcast to all room members and returned by the history endpoint
pub struct ChatMessage {
    pub id: Uuid,
    pub sender_id: Uuid,
    pub sender_role: String,  // e.g. "audience_user", "root_moderator"
    pub content: String,
    pub created_at: DateTime<FixedOffset>,
}

// History endpoint response
pub struct MessageHistoryResponse {
    pub messages: Vec<ChatMessage>,
}

// Unread summary endpoint response
pub struct UnreadSummaryResponse {
    pub questions: Vec<QuestionUnreadCount>,
    pub total_unread: u64,
}

pub struct QuestionUnreadCount {
    pub question_id: Uuid,
    pub unread: u64,
}
```

---

## Backend — Routes

**File:** `src/routes/messages.rs`

All routes are nested under `/api` in `main.rs`.

```
GET  /api/ws/projects/{project_id}/participants/{participant_id}/questions/{question_id}/get
     → WebSocket upgrade

GET  /api/projects/{project_id}/participants/{participant_id}/questions/{question_id}/history
     → Message history (REST)

GET  /api/projects/{project_id}/participants/{participant_id}/unread
     → Unread count summary (REST, admin only)
```

> **Note:** `{participant_id}` in all three routes must be the participant's **account UUID** (`accounts.id`), not the `accounts_projects` row ID.

---

## Backend — WebSocket Handler

**File:** `src/handlers/messages.rs` — `ws_handler` + `handle_socket`

### Authentication

WebSocket cannot use the `Authorization` header (browser limitation). Auth is done via query param:

```
?token=<full_jwt>
```

The token is validated by `verify_token()`. Tokens with a `step` claim (partial auth tokens) are rejected.

### Access Control

| Role | Rule |
|---|---|
| `audience_user` | Can only connect to their own room (`claims.sub == participant_id`). Must be enrolled in the project. |
| `moderator` | Must be assigned to the project in `accounts_projects`. |
| `super_moderator` / `root_moderator` | No additional check — can connect to any room. |

### Socket Lifecycle (`handle_socket`)

Once the upgrade is accepted, the socket is split into independent send/receive halves:

```rust
let (mut ws_sender, mut ws_receiver) = socket.split();
```

Two concurrent tasks are spawned:

**send_task** — listens on the broadcast channel and forwards messages to this client:
```rust
loop {
    match rx.recv().await {
        Ok(json) => ws_sender.send(Message::Text(json.into())).await?,
        Err(RecvError::Closed) => break,
        Err(RecvError::Lagged(_)) => continue,
    }
}
```

**recv_task** — reads messages from this client, saves to DB, and broadcasts:
```rust
while let Some(Ok(msg)) = ws_receiver.next().await {
    match msg {
        Message::Text(text) => {
            if let Ok(incoming) = serde_json::from_str::<IncomingMessage>(&text) {
                save_and_broadcast(...).await;
            }
        }
        Message::Close(_) => break,
        _ => {}
    }
}
```

Both tasks are cancelled via `tokio::select!` as soon as either one finishes (e.g. client disconnects).

### save_and_broadcast

1. Generates a new UUID and timestamp.
2. Inserts the message into the `messages` table.
3. Serializes the message as `ChatMessage` JSON.
4. Calls `tx.send(json)` — broadcasts to all subscribers of this room's channel.

---

## Backend — REST Endpoints

### GET `.../history` — Message History

- Requires `Authorization: Bearer <token>` header.
- Returns all messages for the given `(question_id, participant_id)` room, ordered by `created_at` ascending.
- **Side effect:** marks all messages sent by the *other party* as read (sets `read_at = NOW()`).
- Participants can only fetch their own room. Admins can fetch any room.

**Response:**
```json
{
  "messages": [
    {
      "id": "uuid",
      "sender_id": "uuid",
      "sender_role": "audience_user",
      "content": "Hello!",
      "created_at": "2026-04-15T10:00:00+00:00"
    }
  ]
}
```

### GET `.../unread` — Unread Count Summary

- Admin only (returns 403 for `audience_user`).
- Counts messages sent **by the participant** (where `sender_id == participant_id`) that have `read_at IS NULL`.
- Groups counts by `question_id`.

**Response:**
```json
{
  "questions": [
    { "question_id": "uuid", "unread": 3 }
  ],
  "total_unread": 3
}
```

---

## Backend — State & Broadcast Rooms

**File:** `src/state.rs`

```rust
pub struct AppState {
    pub chat_rooms: Arc<DashMap<(Uuid, Uuid), broadcast::Sender<String>>>,
    // ...
}
```

- `DashMap` key: `(question_id, participant_id)` — a room is identified by the question and the participant.
- Value: a `tokio::sync::broadcast::Sender<String>` with capacity 64.
- Created lazily on first connection via `entry().or_insert_with(|| broadcast::channel(64).0)`.
- Every client that connects to the same room gets a `tx.subscribe()` receiver and shares the same sender.
- Rooms persist for the lifetime of the server process (across connections).

---

## Web Frontend

**Files:**
- `src/services/messageService.ts` — API + WS URL helpers
- `src/pages/ProjectDetail.tsx` — Insights tab with chat panel

### messageService.ts

```typescript
// Load chat history for a room
getMessageHistory(projectId, participantId, questionId): Promise<ChatMessage[]>
// Uses relative URL /api/... → goes through Vite proxy

// Build the WebSocket URL
buildWsUrl(projectId, participantId, questionId): string
// Uses VITE_API_BASE_URL (set to http://localhost:3000 in .env.development)
// to bypass Vite's proxy and connect directly to the backend.
// In production the env var is unset and it falls back to same-origin.
// ws://localhost:3000/api/ws/projects/{p}/participants/{p}/questions/{q}/get?token=...

// Unread count summary for a participant (admin use)
getUnreadSummary(projectId, participantId): Promise<UnreadSummaryResponse>
```

### Why direct WS URL (not Vite proxy)?

Vite's dev proxy (`ws: true`) proxies the WebSocket upgrade correctly but does not reliably forward data frames. The fix is to set `VITE_API_BASE_URL=http://localhost:3000` in `.env.development` so `buildWsUrl` builds an absolute `ws://localhost:3000/...` URL that bypasses the proxy entirely.

### Insights Tab Chat Flow

1. Admin selects a participant from the left sidebar.
2. For each question in each task, a "Chat" button appears.
3. Clicking "Chat" calls `openChat(questionId)`:
   - Loads history via REST (`getMessageHistory`) using `insightParticipant.account_id`.
   - Opens a WebSocket using `buildWsUrl` with `insightParticipant.account_id`.
   - Sets up `onmessage` to append incoming messages to state.
4. Admin types a message and presses Send / Enter:
   - `sendChatMessage()` checks `ws.readyState === WebSocket.OPEN`.
   - Sends `JSON.stringify({ content: "..." })` over the WebSocket.
   - Clears the input field.
   - Message appears when the server echoes it back via broadcast.

### Important: use account_id, not participant_id

`ParticipantOverviewItem` has two ID fields:

| Field | Value | Use for |
|---|---|---|
| `participant_id` | `accounts_projects.id` — the enrollment row ID | UI comparisons, deactivate/detail calls |
| `account_id` | `accounts.id` — the user's actual UUID | **All chat calls (WS URL, history, unread)** |

The WS room key and all DB operations use `accounts.id`. Using `participant_id` (the enrollment row ID) would:
- Connect to the wrong broadcast room (different from the mobile).
- Cause a FK constraint failure on `messages.participant_id → accounts.id`, silently preventing any messages from being saved or broadcast.

---

## Mobile Frontend

**Files:**
- `src/services/messageService.ts` — API + WS URL helpers
- `src/screens/ChatScreen.tsx` — full chat screen
- `src/screens/TaskDetailScreen.tsx` — Chat button per question
- `src/navigation/AppNavigator.tsx` — Chat screen route

### messageService.ts

```typescript
// Load history (same as web but uses API_BASE_URL absolute URL)
getMessageHistory(projectId, participantId, questionId): Promise<ChatMessage[]>

// Build the WebSocket URL (async because token fetch is async)
buildWsUrl(projectId, participantId, questionId): Promise<string>
// → `${API_BASE_URL}/api/ws/projects/{p}/participants/{p}/questions/{q}/get?token=...`

// Decode the JWT sub claim to get the current user's account UUID
getMyParticipantId(): Promise<string>
// Splits the JWT, base64-decodes the payload, returns payload.sub
```

### ChatScreen.tsx

On mount (`useEffect`):
1. Calls `getMyParticipantId()`, `getMessageHistory()`, and `buildWsUrl()` in parallel.
2. Sets message history into state.
3. Opens the WebSocket.
4. Sets up `onopen`, `onclose`, `onerror`, `onmessage` handlers.

On send:
```typescript
ws.send(JSON.stringify({ content: input.trim() }))
```

Messages are identified as "mine" or "theirs" by comparing `item.sender_id === myId` (where `myId` came from `getMyParticipantId()`).

### Navigation

`TaskDetailScreen` has a Chat button on each question card. It navigates to:
```typescript
navigation.navigate('Chat', {
    projectId,
    participantId,   // from getMyParticipantId() — the account UUID
    questionId,
    questionDescription,
})
```

---

## Auth Flow for WebSocket

Standard JWT bearer auth cannot be used for WebSocket upgrades (the browser's WebSocket API doesn't support custom headers on the initial handshake). The token is passed as a query parameter instead:

```
ws://host/api/ws/projects/{p}/participants/{p}/questions/{q}/get?token=<jwt>
```

The backend reads it via `Query(query): Query<WsQuery>` and calls `verify_token()` before the upgrade. If the token is invalid or has the wrong shape, the upgrade is rejected with an HTTP error before any WebSocket connection is established.

---

## Room Key — Critical Detail

The broadcast room is keyed by:

```rust
let room_key = (question_id, participant_id);
//                             ↑
//                   This MUST be accounts.id (the user's account UUID)
//                   NOT accounts_projects.id (the enrollment row UUID)
```

Both the admin (web) and the participant (mobile) must use the same UUID here or they end up in separate rooms and can never communicate.

- **Mobile:** `participantId` = JWT `sub` = `accounts.id` ✓
- **Web:** must use `insightParticipant.account_id`, not `insightParticipant.participant_id` ✓

---

## Message Flow Diagram

```
PARTICIPANT (mobile)                  BACKEND                        ADMIN (web)
      |                                  |                                |
      |-- WS connect ?token=... -------->|                                |
      |                                  |-- subscribe(room_key) -------->|
      |                                  |<-- WS connect ?token=... ------|
      |                                  |
      |-- send { content: "hello" } ---->|
      |                   recv_task receives text frame
      |                   save_and_broadcast():
      |                     1. INSERT INTO messages (...)
      |                     2. tx.send(json)  ← broadcasts to all rx
      |                                  |
      |<-- onmessage (echo) -------------|-- onmessage (delivery) ------->|
      |    message appears in mobile     |    message appears in web      |
      |                                  |                                |
      |                          (and vice versa)
```

Every message is:
1. Received by the server from one client.
2. Saved to the database permanently.
3. Broadcast to ALL clients subscribed to that room — including the original sender (echo).
4. Displayed in the UI via the `onmessage` handler on each client.
