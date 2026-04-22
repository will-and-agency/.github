# Frontend — Notifications & Automatic Project Statuses

Covers changes made to **MobileEtnoV2_frontendweb** (React/Vite web app) and **MobileEtnoV2_frontendapp** (React Native/Expo mobile app).

---

## Part 1 — Automatic Project Statuses

### What changed and why

Previously, admins could manually change a project's status using a dropdown on the project card. This caused 400 errors for seeded/old projects (whose `ends_at` dates were in the past), which also blocked cover image uploads since they were chained after the failing update call.

Status is now **computed by the backend on every read** based on the project's dates:

| Status | When |
|---|---|
| `draft` | `starts_at` is in the future |
| `active` | No `starts_at`, or `starts_at` already passed, and `ends_at` is future |
| `archived` | `ends_at` has passed |

The frontend never sends a status value — it just displays whatever the backend returns.

`completed` status has been removed entirely.

---

### Web — `src/pages/Projects.tsx`

**Removed:**
- `updateProject` import (was only used by the status dropdown)
- The `<select>` status dropdown from each project card
- The `onChange` handler that called `updateProject` with a new status
- The `setProjectsTab('archived')` / `else if` logic that auto-switched tabs when archiving manually

**Added:**
- A plain `<span>` badge in place of the dropdown:
```tsx
<span className={getBadgeClass(project.status)}>
    {project.status ?? 'draft'}
</span>
```

The archived tab still works exactly as before — it filters on `p.status === 'archived'`, which the backend now sets automatically when `ends_at` passes.

**Minor leftover (harmless):** `getBadgeClass` still has a `'completed'` case that will never match. It can be cleaned up but causes no issues.

---

### Web — `src/pages/ProjectDetail.tsx`

**Removed from `openEditProject`:**
```ts
setProjFormStatus(project.status ?? 'active')
```

**Removed from `handleEditProject`:**
```ts
status: projFormStatus || undefined,
```

**Removed from `ProjectForm` component JSX:**
The entire status `<div className="form-group">` block containing the `<select>` with Active / Draft / Archived / Completed options was removed from the form. The `status` and `setStatus` props were also removed from the component signature.

The `projFormStatus` and `setProjFormStatus` state variables were removed.

---

### Web — `isArchived` lifted to component level

Previously `isArchived` was computed inside a JSX IIFE (immediately invoked function expression) which made it invisible to event handlers like `openChat`. It is now declared at the top of the component:

```tsx
const isArchived = project?.status === 'archived'
```

This single boolean is used throughout the component to gate archived-specific behaviour (read-only banner, hiding action buttons, disabling chat, etc).

---

### Web — Archived project: Insights tab & read-only chat

The insights tab is fully accessible for archived projects. The only restriction is the chat panel.

**`openChat` function:** For archived projects the WebSocket connection is **not opened**. Message history is still fetched normally via `getMessageHistory`. This prevents spurious WS errors in the console and makes `chatConnected` correctly stay `false`.

```ts
if (!isArchived) {
    const ws = new WebSocket(wsUrl)
    // ... attach handlers
    chatWsRef.current = ws
}
```

**Chat panel input row:** For archived projects the input and Send button are replaced with a read-only notice:

```tsx
{isArchived ? (
    <p style={{ ... }}>Project archived — chat is read-only.</p>
) : (
    <div className="chat-input-row">
        <input ... />
        <button>Send</button>
    </div>
)}
```

Message history, the Answer card, and the Notes tab are all still fully visible.

---

### Web — Per-participant answer counts fixed

In the Insights → Participant sub-tab, each question row shows a progress indicator of how many answered that question. Previously it showed `(total answers from all participants) / (total enrolled participants)`, which was wrong for a per-participant view.

**Fix in `selectInsightParticipant`:**
```ts
// Before — counted all answers regardless of who submitted them
answerCounts[q.id] = ans.length

// After — checks only if the selected participant answered
answerCounts[q.id] = ans.some(a => a.user_id === p.account_id) ? 1 : 0
```

**Fix in the render:**
```tsx
// Before
const total = participants.length  // all enrolled participants

// After
const total = 1  // always 1: did this participant answer or not
```

The progress donut now shows full green (answered) or empty red (not answered). The fraction label was replaced with plain text: `"Answered"` or `"Not answered"`.

---

### Mobile app — `src/screens/ProjectsScreen.tsx`

Archived projects are shown greyed out and cannot be tapped to navigate into.

**Changes to `renderItem`:**

```tsx
const isArchived = item.status === 'archived'

<TouchableOpacity
    style={[styles.card, isArchived && styles.cardArchived]}
    activeOpacity={isArchived ? 1 : 0.7}
    onPress={() => {
        if (isArchived) return
        navigation.navigate('ProjectDetail', { ... })
    }}
>
```

- `style={[styles.card, isArchived && styles.cardArchived]}` — applies `opacity: 0.45` to the whole card including the cover image
- `activeOpacity={isArchived ? 1 : 0.7}` — archived cards show no press feedback (no dimming on tap)
- `if (isArchived) return` — prevents navigation even if the press fires

Text, badge, and date label all receive a muted grey style when archived:
```tsx
<Text style={[styles.company, isArchived && styles.textArchived]}>
<Text style={[styles.name, isArchived && styles.textArchived]}>
<Text style={[styles.description, isArchived && styles.textArchived]}>
<Text style={[styles.date, isArchived && styles.textArchived]}>
```

The status badge also changes colour:
```tsx
<View style={[styles.statusBadge, isArchived && styles.statusBadgeArchived]}>
    <Text style={[styles.statusText, isArchived && styles.statusTextArchived]}>
        {item.status}
    </Text>
</View>
```

**New styles added:**
```ts
cardArchived:       { opacity: 0.45 }
textArchived:       { color: '#9CA3AF' }
statusBadgeArchived: { backgroundColor: '#F3F4F6' }
statusTextArchived: { color: '#6B7280' }
```

---

## Part 2 — Notifications

### Web — `src/services/attachmentService.ts`

No notification-specific changes here, but this file is relevant context: the service handles the three-step upload flow (init → upload to S3 → confirm). This was already in place; no changes were needed for notifications.

---

### Web — Notification WebSocket

The web app connects to `GET /api/ws/notifications?token=<jwt>` on load. This is a persistent WebSocket that the backend uses to push real-time signals when new chat messages arrive.

**Important:** The WebSocket failure message seen in the console (`WebSocket is closed before the connection is established`) is caused by React Strict Mode mounting components twice in development. It is a dev-only artefact and does not happen in production.

---

### Mobile app — Push notifications (`src/screens/ProjectsScreen.tsx`)

On mount, `ProjectsScreen` requests notification permissions and registers the device's Expo push token with the backend (`POST /api/push-tokens`). This token allows the backend to send native push notifications to the device even when the app is in the background.

The setup only runs on real devices (`Device.isDevice` check) — simulators/emulators cannot receive push notifications.

An Android notification channel named `"default"` with maximum importance is created on Android to ensure notifications are shown with sound and vibration.

---

### Web — Notification bell icon

A bell icon appears in the top-right of `ProjectDetail` (and other screens that show it). It shows an unread count badge if there are unread notifications.

- The count is fetched via `GET /api/participant/notifications` (for participants) or `GET /api/notifications` (for admins) on every screen focus
- Tapping the bell navigates to the Notifications screen
- The badge shows `9+` for counts above 9

---

### Console errors explained

When the app first loads you may see these in the browser console — here is what each one means:

| Error | Cause | Action needed |
|---|---|---|
| `WebSocket closed before connection established` | React Strict Mode double-mount in dev | None — does not happen in production |
| `404 /api/projects/{id}/attachments/cover-image` | Project has no cover image uploaded | None — frontend handles this gracefully (shows no image) |
| `400 PUT /api/projects/update/{id}` | **Fixed** — was caused by past `ends_at` on seeded projects + manual status dropdown | No longer happens; status dropdown removed |
