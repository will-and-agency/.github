# ProjectDetail & TaskDetail Refactor

## Overview

`ProjectDetail.tsx` (1,382 lines) and `TaskDetail.tsx` (1,111 lines) were broken up into 12 focused component files. Both pages had grown to contain inline modals, form components, and UI helpers at the bottom of the file. The refactor extracts all of these into their own files, leaving the pages responsible only for data fetching, state coordination, and layout.

---

## What Was Extracted

### Shared UI — `src/components/`

| File | What it is |
|---|---|
| `Modal.tsx` | Overlay/card shell used by every modal. Accepts `title`, `onClose`, `children`. |
| `StarRating.tsx` | 5-star interactive rating widget used in the participants table. |
| `InfoCell.tsx` | Labeled value box (label on top, value below) used in the participant detail modal. |

### Form Components — `src/components/forms/`

| File | What it is |
|---|---|
| `TaskForm.tsx` | Create/edit form for a task: title, description, sort order, JSON config, cover image, intro video, prep materials. |
| `ProjectForm.tsx` | Edit form for a project: name, company, description, start/end dates, cover image. |

### Participant Modals — `src/components/participants/`

| File | What it is |
|---|---|
| `CsvUploadModal.tsx` | Uploads a participant CSV. Runs the full S3 three-step flow internally. Shows added/skipped counts and per-row remove buttons. |
| `InviteModal.tsx` | Sends invitation emails to all uninvited participants. Fields: reward amount, reward deadline, manager email, language. |
| `AddParticipantModal.tsx` | Adds a single participant by email with optional metadata key-value rows. |
| `ParticipantDetailModal.tsx` | Shows a participant's full profile and has a Deactivate button. Fetches its own data via `useEffect`. |

### Question Components — `src/components/questions/`

| File | What it is |
|---|---|
| `QuestionForm.tsx` | Create/edit form for a question: description, type, sort order, required toggle. Owns the `QUESTION_TYPES` constant. |
| `OptionsEditor.tsx` | Manages answer options for single/multi-choice and ranking questions. Add, delete, upload image per option. |
| `ScaleConfigEditor.tsx` | Configures a scale question: min/max, labels, shape, color, cumulative mode. Owns `SCALE_PRESETS`, `SHAPE_OPTIONS`, and the symbol maps. Includes a live preview. |

---

## ProjectDetail Changes

**Before:** 1,382 lines  
**After:** 834 lines

### Imports removed
- `useRef` — no longer needed (fileInputRef moved into `CsvUploadModal`)
- `Link` — unused
- `getParticipantDetail`, `uploadParticipantCsv`, `sendInvites`, `addParticipantManually` — logic moved into the modal components
- `ParticipantDetail`, `UploadCsvResult`, `SendInvitesResult` — types only needed inside the modals

### State removed (17 variables)
All participant modal state, CSV upload state, and invite state moved into their respective self-contained modal components:

```
selectedParticipant    participantLoading    participantError
addMetaRows            addEmail
csvFile                uploadStep            uploadError
uploadResult           fileInputRef
rewardAmount           rewardDeadline        managerEmail
language               inviteStep            inviteError
inviteResult
```

### State added (1 variable)
```ts
const [selectedParticipantId, setSelectedParticipantId] = useState<string | null>(null)
```

### Handlers replaced
The async `openParticipantDetail` (which fetched participant data inline), `handleDeactivate`, `handleUploadCsv`, `handleSendInvites`, and the modal-opening functions that reset a dozen state variables were all replaced with four one-liners:

```ts
const openParticipantDetail = (participantId: string) => {
    setSelectedParticipantId(participantId); setParticipantModal('detail')
}
const openCsvModal    = () => setParticipantModal('csv')
const openInviteModal = () => setParticipantModal('invite')
const openAddModal    = () => setParticipantModal('add')
```

### Participant modal JSX replaced
~300 lines of inline modal JSX replaced with 4 component calls:

```tsx
{participantModal === 'csv' && (
    <CsvUploadModal projectId={projectId!} onClose={() => setParticipantModal(null)} onParticipantsChanged={reloadParticipants} />
)}
{participantModal === 'invite' && (
    <InviteModal projectId={projectId!} onClose={() => setParticipantModal(null)} />
)}
{participantModal === 'add' && (
    <AddParticipantModal projectId={projectId!} onClose={() => setParticipantModal(null)} onAdded={reloadParticipants} />
)}
{participantModal === 'detail' && selectedParticipantId && (
    <ParticipantDetailModal
        projectId={projectId!}
        participantId={selectedParticipantId}
        onClose={() => { setParticipantModal(null); setSelectedParticipantId(null) }}
        onDeactivated={reloadParticipants}
    />
)}
```

### Bottom helpers removed
`StarRating`, `InfoCell`, `TaskForm`, `ProjectForm`, and `Modal` were defined inline at the bottom of the file (~200 lines). These are now imported from their component files.

---

## TaskDetail Changes

**Before:** 1,111 lines  
**After:** 758 lines

### Imports added
```ts
import Modal            from '../components/Modal'
import QuestionForm     from '../components/questions/QuestionForm'
import OptionsEditor    from '../components/questions/OptionsEditor'
import ScaleConfigEditor from '../components/questions/ScaleConfigEditor'
```

### Constants removed
`QUESTION_TYPES` (9 entries) and `SCALE_PRESETS` (3 entries) were defined at module level but only used inside the inline component definitions. They now live inside `QuestionForm.tsx` and `ScaleConfigEditor.tsx` respectively.

`STRUCTURED_TYPES` remains in `TaskDetail` because it is used in the component's own logic (`openEdit`, `handleCreate`, and the modal JSX conditions).

### Bottom section removed
The 354-line block of inline function definitions (`QuestionForm`, `OptionsEditor`, `ScaleConfigEditor`, `Modal`) was deleted. The JSX in the return already referenced these by name — no other changes were needed in the modal rendering code since it was already written to use them as components.

---

## Design Principle

The participant modals (`CsvUploadModal`, `InviteModal`, `AddParticipantModal`, `ParticipantDetailModal`) are **self-contained**: they own their own state and make their own API calls. The parent only passes `projectId` and callback props (`onClose`, `onAdded`, etc.). This is intentional — it means adding a new place in the app that needs to open one of these modals requires zero state work in the parent.

The form components (`TaskForm`, `ProjectForm`, `QuestionForm`) are **controlled**: all state lives in the parent and is passed down as props. This is also intentional — the parent (ProjectDetail or TaskDetail) coordinates multiple pieces of state that the form needs to interact with (e.g. cover image files, loading flags, submit handlers tied to create vs. edit mode).
