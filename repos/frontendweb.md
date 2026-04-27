# MobileEtnoV2 — Web Frontend Overview

> **Who this is for:** Anyone who needs to understand, run, or deploy the web app.
> This document explains everything from scratch — no assumptions.
> **Last updated:** 2026-04-27 (after merging the `feat/mcq` + insights branch)

---

## What is this?

This is the **admin web application** for the MobileEtno platform. It runs in a desktop browser.

**Who uses it?** Researchers and moderators — not participants. Participants use the mobile app.

**What can you do here?**
- Log in (with optional 2-factor authentication)
- Create and manage research projects (including archiving)
- Create tasks and survey questions inside projects
- Upload participants via CSV and send invitation emails
- View participant progress, answers, and ratings
- Chat in real-time with participants directly from the insights panel
- Write private notes on participant answers
- View analytics charts (completion status, answers by participant, per-task activity)
- Manage moderator accounts (create, suspend, delete)
- Assign moderators to projects

---

## Tech Stack — What Technologies are Used?

| Technology | Version | What it does |
|---|---|---|
| **React** | 19.2 | UI library. Builds the interface from components. |
| **TypeScript** | 6.0 | Adds types to JavaScript so errors are caught before runtime. |
| **Vite** | 8.0 | Build tool and local dev server. Very fast. |
| **React Router** | 7.13 | Handles client-side navigation (URL changes without page reloads). |
| **Axios** | 1.13 | Sends HTTP requests to the backend API. |
| **Recharts** | 3.8 | **New.** Renders bar charts, pie charts, and line charts in the Insights tab. |
| **jwt-decode** | 4.0 | Decodes JWT tokens to check expiry on the frontend. |
| **qrcode.react** | 4.2 | Renders a QR code for the 2FA setup screen. |

**No external UI library** — all styling is custom CSS (see `src/index.css`).

**No global state management library** — React's built-in `useState` is used everywhere.

---

## Folder Structure — What Does Each File Do?

```
MobileEtnoV2_frontendweb/
│
├── src/
│   │
│   ├── main.tsx                ← Entry point. Mounts the React app into index.html.
│   ├── App.tsx                 ← Defines all URL routes.
│   ├── Router.tsx              ← (empty — routing is done inside App.tsx)
│   ├── index.css               ← All global CSS styles (includes new insight panel styles).
│   ├── vite-env.d.ts           ← TypeScript types for Vite env vars.
│   │
│   ├── pages/                  ← One file = one full page of the app.
│   │   ├── Login.tsx                   ← The login form.
│   │   ├── ChangeInitialPassword.tsx   ← Forces a password change on first login.
│   │   ├── SetupTwoFactor.tsx          ← Shows QR code to set up authenticator app.
│   │   ├── VerifyTwoFactor.tsx         ← Accepts the 6-digit TOTP code.
│   │   ├── Projects.tsx                ← Lists all projects. Has All/Archived tabs.
│   │   ├── CreateProject.tsx           ← Form to create a new project.
│   │   ├── ProjectDetail.tsx           ← ★ Heavily refactored. Now has 4 tabs (see below).
│   │   ├── TaskDetail.tsx              ← Shows questions inside a task. Manages them.
│   │   └── AdminEdit.tsx               ← Manage moderator accounts (root/super only).
│   │
│   ├── components/             ← Reusable building blocks used by multiple pages.
│   │   ├── Layout.tsx          ← The shell around every page. Now has live notification bell.
│   │   ├── PrivateRoute.tsx    ← Guards protected pages (redirects to login if not logged in).
│   │   ├── InsightsTab.tsx     ← ★ New. Top-level insights tab with 3 sub-tabs.
│   │   ├── ProgressDonut.tsx   ← ★ New. Tiny donut graphic showing answer completion ratio.
│   │   ├── TaskCharts.tsx      ← ★ New. AI video analysis charts (facial/speech/text emotion).
│   │   └── insights/           ← ★ New folder. Three sub-components of InsightsTab.
│   │       ├── ProjectInsights.tsx     ← Project-wide charts (completion pie, answers bar, activity by task).
│   │       ├── TaskInsights.tsx        ← Per-task view: questions, all answers, AI analysis placeholder.
│   │       └── ParticipantInsights.tsx ← Per-participant view: questions, inline chat + notes panel.
│   │
│   ├── components/sample_data/ ← ★ New. Test/sample data for TaskCharts development.
│   │   ├── testProjectData.ts          ← Hardcoded emotion data for chart prototyping.
│   │   ├── charts.py                   ← Original Python Bokeh chart code (reference only).
│   │   ├── test project.csv            ← Sample respondent CSV.
│   │   └── test project_task 1_...csv  ← Temporal emotion CSVs for line chart.
│   │
│   ├── services/               ← Functions that call the backend API.
│   │   ├── authService.ts      ← Login, change password, setup/verify 2FA.
│   │   ├── projectService.ts   ← Get, create, update, delete projects.
│   │   ├── taskService.ts      ← Get, create, update, delete tasks.
│   │   ├── questionService.ts  ← Get, create, update, delete questions.
│   │   ├── questionOptionService.ts ← Get, create, update, delete question options.
│   │   ├── answerService.ts    ← Get answers from participants.
│   │   ├── attachmentService.ts ← File upload (init → S3 → confirm) for project/task/question/option.
│   │   ├── messageService.ts   ← Chat history, WebSocket URL builder, notifications, dismiss.
│   │   ├── noteService.ts      ← Admin notes on answers (create, read, update, delete).
│   │   ├── participantService.ts ← Participant management, ratings, CSV upload, send invites.
│   │   └── adminService.ts     ← Moderator account management.
│   │
│   └── types/                  ← TypeScript type definitions.
│       ├── auth.ts             ← LoginRequest, LoginResponse, etc.
│       ├── project.ts          ← Project, CreateProjectRequest, UpdateProjectRequest.
│       ├── task.ts             ← Task, TaskRequest.
│       ├── question.ts         ← Question, QuestionType, ScaleConfig, QuestionOption, etc.
│       └── admin.ts            ← Account, CreateModeratorRequest, UpdateModeratorRequest.
│
├── index.html                  ← The one HTML file. Vite injects the JS bundle here.
├── vite.config.ts              ← Vite config (proxy /api/* to localhost:3000 in dev, WebSocket support).
├── tsconfig.json               ← TypeScript compiler settings.
├── package.json                ← Dependencies and npm scripts.
├── .env.development            ← VITE_API_BASE_URL for local development.
└── dist/                       ← Built output (created by `npm run build`).
    ├── index.html
    └── assets/                 ← Bundled JS and CSS files.
```

---

## All Pages — What Each Page Does

### Public Pages (no login needed)

| URL | Page | What it does |
|---|---|---|
| `/auth/login` | `Login.tsx` | Email + password form. Calls `POST /api/auth/web/login`. |
| `/auth/change-initial-password` | `ChangeInitialPassword.tsx` | Forces you to set a real password on first login using a temporary password received by email. |
| `/auth/setup-2fa` | `SetupTwoFactor.tsx` | Calls `POST /api/auth/setup-2fa`, displays a QR code to scan with Google Authenticator. |
| `/auth/verify-2fa` | `VerifyTwoFactor.tsx` | You enter the 6-digit code from your authenticator app to complete login. |
| `/*` (anything else) | Redirect | All unknown URLs redirect to `/auth/login`. |

### Protected Pages (login required)

These pages are wrapped in `<PrivateRoute>`. If your JWT token is missing or expired, you get redirected to `/auth/login` automatically.

| URL | Page | What it does |
|---|---|---|
| `/projects` | `Projects.tsx` | Lists all your projects. Has an "All Projects" tab and an "Archived" tab. Shows cover images. Inline project deletion. |
| `/projects/create` | `CreateProject.tsx` | Form to create a new project (name, company, description, start/end dates, optional cover image). |
| `/projects/:projectId` | `ProjectDetail.tsx` | ★ The main project workspace. Has 4 tabs (see detailed breakdown below). |
| `/projects/:projectId/tasks/:taskId` | `TaskDetail.tsx` | Shows all questions in a task. Add, edit, delete questions. Supports 8 question types. |
| `/admin` | `AdminEdit.tsx` | Manage moderator accounts. Root can manage everyone; Super can create standard moderators. |

---

## ProjectDetail — The 4-Tab Workspace

`ProjectDetail.tsx` is the most important page. It now has **four tabs**, and the app remembers which tab you were on (stored in `localStorage` per project).

### Tab 1: Participants

What you see here:
- A table of all participants in the project
- Each row shows: email, completion status (not started / in progress / completed), number of questions answered out of total, average star rating
- A small donut chart per participant showing their completion
- Actions: view participant detail, deactivate participant, set/update star rating (1–5)

What you can do:
- **Upload CSV** → a modal lets you upload a `.csv` file with participant emails. The file is uploaded to S3 first, then the backend processes it and creates accounts.
- **Send invites** → a modal lets you set reward amount, reward deadline, manager email, and language (Danish/English). Sends invitation emails to all participants who haven't been invited yet.

### Tab 2: Insights ★ New

The analytics and research insight panel. It has **3 sub-tabs**:

#### Insights → Project
Project-wide overview charts (powered by Recharts):
- **Participant completion status** — donut/pie chart showing how many participants are: Not started / In progress / Completed
- **Answers by participant** — stacked bar chart of answered vs. remaining questions, top 8 participants
- **Activity by task** — bar chart showing answered and total questions per task

#### Insights → Task
Per-task analysis view:
- **Sidebar**: list of all tasks — click one to load its data
- **Main area**: for the selected task, shows a dropdown to filter by question, then for each question:
  - All participant answers (text shown inline, MCQ shown as chips, media shown as "Media answer submitted")
  - For **video** answers: a "Show AI Analysis" toggle that expands `TaskCharts` — this shows temporal emotion line charts, valence/arousal donuts using pre-loaded sample CSV data (placeholder for real AI integration)
  - **AI Overview Summary**, **Key Themes**, and **Actionable Insight** placeholder cards

#### Insights → Participant
Per-participant deep-dive (most feature-rich):
- **Sidebar**: list of all participants — click one to load their view
- **Main area**: for the selected participant, shows every task and every question, with:
  - Question type badge
  - Answer completion count (e.g., "3/10 answered") with a `ProgressDonut` graphic
  - Clicking a question row expands an **inline panel** with two side-by-side cards:
    - **Answer card**: shows the participant's actual answer (text, MCQ chips, or "media submitted")
    - **Chat/Notes card** (tabbed):
      - **Chat tab**: real-time WebSocket chat between this moderator and this participant, specific to this question. Shows connected/disconnected status. You can type and send messages.
      - **Notes tab**: private moderator notes linked to this answer. You can add, edit, and delete notes.

### Tab 3: Tasks

Task management for this project:
- Lists all tasks sorted by their `sort_order` number
- Shows cover images (if uploaded)
- Create, edit, and delete tasks via modals
- Edit modal also lets you upload/replace a task cover image
- Clicking a task navigates to `/projects/:projectId/tasks/:taskId` (the `TaskDetail` page) where you manage that task's questions

### Tab 4: Clipboard

(Visible in the tab bar; content not fully detailed in this document — inspect `ProjectDetail.tsx` for its implementation.)

---

## The Layout Header — Notifications Bell

`Layout.tsx` wraps every protected page. The header now includes:

- **MobileEtno logo** — links back to `/projects`
- **Notification bell** — live badge showing unread message count
  - Opens a dropdown listing all conversations where a moderator has unread messages
  - Each notification shows: project name, participant email, message preview, unread count
  - Clicking a notification marks it as read and navigates to `ProjectDetail` with `state: { tab: 'insights', participantId, questionId }` pre-set — the insights panel opens directly to that participant and question
  - The "×" button dismisses a notification
  - A **WebSocket connection** keeps the bell up to date in real time, reconnecting automatically if it drops
- **Logout button** — clears `full_token` from sessionStorage and redirects to `/auth/login`

---

## How Authentication Works in the Browser

The app uses **two separate tokens**, both stored in `sessionStorage`:

| Key | Token | What it's for |
|---|---|---|
| `temp_token` | Temporary JWT (5 min) | Held during the 2FA flow (between submitting password and verifying TOTP code). |
| `full_token` | Full JWT (24 hours) | Held after successful login. Sent with every API request. |

### Why `sessionStorage` (not `localStorage`)?

`sessionStorage` is cleared automatically when the browser tab is closed. This means closing the tab logs you out automatically — safer than `localStorage` which persists across browser restarts.

**Note:** The insights sub-tab and project detail tab selections ARE stored in `localStorage` (not sessionStorage) so they survive tab closes and are remembered per project.

### How `PrivateRoute` Works

Before rendering a protected page, `PrivateRoute.tsx`:
1. Checks if `full_token` exists in sessionStorage.
2. Decodes the JWT and checks the `exp` (expiry) field.
3. If the token is missing or expired: clears sessionStorage and redirects to `/auth/login`.
4. If the token is valid: renders the page normally.

---

## How API Calls Work

All API calls go to the backend via the `/api` prefix.

**In development:** Vite's proxy (`vite.config.ts`) forwards `/api/*` to `http://localhost:3000` and also proxies WebSocket upgrades (`ws: true`). This means:
- The app runs on `http://localhost:5173`
- When it calls `/api/projects/get`, Vite silently forwards it to `http://localhost:3000/api/projects/get`
- No CORS issues in development

**Important — WebSocket in development:** Vite's WebSocket proxy drops data frames (a known Vite limitation). The `messageService.ts` works around this by connecting WebSockets directly to the backend URL (`VITE_API_BASE_URL`) instead of going through the Vite proxy. In production this variable is unset and the app falls back to same-origin, which works because the backend and frontend share a domain.

**In production:** A reverse proxy (Nginx or similar) must:
- Serve the `dist/` static files for regular requests
- Forward `/api/*` HTTP requests to the backend
- Forward WebSocket upgrade requests for `/api/ws/*` to the backend (requires `Upgrade` and `Connection` headers)

Every authenticated API call includes this header:
```
Authorization: Bearer <full_token from sessionStorage>
```

---

## Question Types

The web app can create survey questions of 8 types:

| Type | What participants see |
|---|---|
| `text` | A text box to type a free answer |
| `video` | They record or upload a video |
| `image` | They take or upload a photo |
| `single_choice` | Radio buttons — pick one option |
| `multi_choice` | Checkboxes — pick multiple options |
| `ranking` | Drag options into an order |
| `scale` | A rating scale (e.g., stars 1–5, NPS 0–10, Likert 1–7). Configurable: min/max, labels, shape (star/circle/heart/diamond), color, cumulative mode. |
| `click_map` | Click on a specific area of an uploaded image |

For **scale** questions, presets are available: NPS (0–10), Stars (1–5), Likert (1–7).

For **single/multi/ranking** questions, you add options (with optional images) after creating the question.

For **click_map** questions, you upload a background image after creating the question.

---

## File Upload Flow

Uploading any file (image, video, CSV) uses a 3-step process to avoid sending large files through the backend:

```
Step 1: Tell the backend you want to upload
        → POST /api/projects/{id}/attachments/init
           (or task/question/option variants)
        ← Server returns: { attachment_id, upload_url, file_key }

Step 2: Upload the file DIRECTLY to S3 using the pre-signed URL
        → PUT <upload_url>  (file bytes, with Content-Type header)
        ← S3 returns 200 OK

Step 3: Tell the backend the upload is done
        → POST /api/projects/{id}/attachments/{attachment_id}/confirm
        ← Server records the file and links it to the entity
```

Upload endpoints available:
- **Project cover image**: `initUpload(projectId, 'cover_image', ...)`
- **Task cover image**: `initTaskUpload(projectId, taskId, 'cover_image', ...)`
- **Click map question image**: `initClickMapImageUpload(projectId, taskId, questionId, ...)`
- **Question option image**: `initOptionUpload(projectId, taskId, questionId, optionId, ...)`

---

## Environment Variables

**For development** (`.env.development`):
```env
VITE_API_BASE_URL=http://localhost:3000
```

This is used by `messageService.ts` to connect WebSockets directly to the backend (bypassing the Vite proxy which drops WebSocket data frames).

**For production:** No `.env` file is needed for the deployed app. Configure Nginx to proxy `/api/*` and WebSocket upgrades to the backend.

---

## How to Run Locally

### Prerequisites

- Node.js 18+

### Steps

```bash
# Step 1: Go into the project folder
cd C:\Projects\MobileEtnoV2_frontendweb

# Step 2: Install dependencies (run this after any pull that changes package.json)
npm install

# Step 3: Make sure the backend is running on port 3000

# Step 4: Start the development server
npm run dev
```

The app will open at `http://localhost:5173`.

Changes to files are reflected instantly (hot module replacement — no need to refresh).

### Other Commands

```bash
# Build for production (output goes to dist/)
npm run build

# Preview the production build locally before deploying
npm run preview
```

---

## Building for Production

```bash
npm run build
```

This runs `tsc` (TypeScript type check) then `vite build`.

The output is in `dist/`:
```
dist/
├── index.html
└── assets/
    ├── index-<hash>.js    ← All JavaScript (bundled + minified)
    └── index-<hash>.css   ← All CSS (bundled + minified)
```

The `<hash>` in filenames changes on every build so browsers always download the latest version instead of using a stale cached version.

---

## Deployment

### What to Deploy

The `dist/` folder — static HTML, JS, and CSS. No Node.js server is needed to host it.

### Nginx Configuration Example

```nginx
server {
    listen 443 ssl;
    server_name yourapp.com;

    root /var/www/mobileetno-web/dist;
    index index.html;

    # Serve the React SPA — fallback to index.html for client-side routes
    location / {
        try_files $uri /index.html;
    }

    # Proxy all API calls to the backend (HTTP + WebSocket upgrades)
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Important:** The `try_files $uri /index.html` line is required. Without it, refreshing the browser on any page other than `/` returns a 404 error.

**Important:** The `Upgrade` and `Connection` headers in the `/api/` block are required for the WebSocket connections (chat and notification bell) to work in production.

### Deployment Steps

```bash
# 1. Build
npm run build

# 2. Copy dist/ to your server
scp -r dist/ user@yourserver:/var/www/mobileetno-web/

# 3. Reload Nginx
ssh user@yourserver "sudo systemctl reload nginx"
```

### Deployment Checklist

- [ ] `npm install` run after pulling (especially important after this merge — `recharts` was added)
- [ ] `npm run build` completes without TypeScript errors
- [ ] `dist/` folder created successfully
- [ ] Nginx configured with `try_files` and WebSocket proxy headers
- [ ] HTTPS enabled (Let's Encrypt via Certbot or Caddy)
- [ ] Test login
- [ ] Test 2FA setup flow
- [ ] Test creating a project and uploading participants
- [ ] Test Insights tab — charts render, chat works in Participant sub-tab
- [ ] Test notification bell — badge updates, clicking a notification navigates correctly

### No CI/CD Yet

There is no automated CI/CD pipeline. Build and deploy manually:

```bash
npm run build
scp -r dist/ user@yourserver:/var/www/mobileetno-web/
```

---

## What Changed in the Latest Merge (2026-04-27)

The merge `c5ac218` ("merged with tomaz") brought in the insights/analytics feature:

| What | Change |
|---|---|
| `recharts` dependency | Added. Required for all charts in the Insights tab. Run `npm install` after pulling. |
| `src/components/InsightsTab.tsx` | **New.** Top-level Insights tab component with Project/Task/Participant sub-tabs. |
| `src/components/ProgressDonut.tsx` | **New.** Small SVG donut showing answered/total per question. |
| `src/components/TaskCharts.tsx` | **New.** Recharts line + donut charts for AI video emotion analysis (currently uses sample data). |
| `src/components/insights/ProjectInsights.tsx` | **New.** Project-wide completion and activity charts. |
| `src/components/insights/TaskInsights.tsx` | **New.** Per-task answer browser with AI analysis placeholder. |
| `src/components/insights/ParticipantInsights.tsx` | **New.** Per-participant view with inline chat and notes. |
| `src/components/sample_data/` | **New.** Sample CSVs and TypeScript test data for TaskCharts (not production data). |
| `src/pages/ProjectDetail.tsx` | **Major refactor.** Now has 4 tabs (participants, insights, tasks, clipboard). Task CRUD moved in-page. Project edit modal added. |
| `src/pages/TaskDetail.tsx` | Minor update. |
| `src/pages/Projects.tsx` | Added Archived tab and inline project deletion. |
| `src/components/Layout.tsx` | Notification bell with real-time WebSocket. Logout button. |

---

## Common Issues

| Problem | Cause | Fix |
|---|---|---|
| Charts not rendering after pull | `recharts` not installed | Run `npm install` |
| Blank page after refreshing on any route | Nginx missing `try_files` | Add `try_files $uri /index.html` to Nginx config |
| Chat and notifications don't work in production | Nginx not proxying WebSocket upgrades | Add `Upgrade` and `Connection` headers to the `/api/` proxy block |
| Session lost on tab close | Expected — `sessionStorage` is cleared on tab close | Normal behaviour by design |
| Insights tab selection not remembered | Check if `localStorage` is blocked | The tab preference uses `localStorage`; ensure it's not blocked |
| WebSocket connects but messages don't arrive in dev | Vite proxy drops WebSocket data frames | Check that `VITE_API_BASE_URL` is set in `.env.development` — the WS URL builder uses it to bypass the proxy |
