# MobileEtnoV2 — Mobile App Overview

> **Who this is for:** Anyone who needs to understand, run, or build the mobile app.
> This document explains everything from scratch — no assumptions.

---

## What is this?

This is the **participant mobile app** for the MobileEtno platform. It runs on iOS and Android phones.

**Who uses it?** Research participants — not researchers. Researchers use the web app.

**What can participants do?**
- Log in with the credentials they received by email
- Change their password on first login
- Browse research projects they've been assigned to
- View tasks within a project and answer survey questions
- Upload photo and video answers
- Chat in real-time with a moderator about a specific question
- Receive push notifications when a moderator sends them a message

---

## Tech Stack — What Technologies are Used?

| Technology | Version | What it does |
|---|---|---|
| **React Native** | 0.81.5 | Lets us write the app in JavaScript/TypeScript and run it on both iOS and Android. |
| **Expo** | SDK 54 | A toolkit on top of React Native that simplifies builds, native modules, and updates. |
| **TypeScript** | 5.9 | Adds types to JavaScript to catch errors early. |
| **React Navigation** | 7 (native stack) | Handles moving between screens (like going from Projects to a specific Project). |
| **Axios** | 1.15 | Sends HTTP requests to the backend API. |
| **AsyncStorage** | 2.2 | Stores data on the phone (like the login token) so the app remembers you between sessions. |
| **expo-notifications** | 0.32 | Handles push notifications (receiving and displaying them). |
| **expo-file-system** | 19 | Uploads files (photos, videos) directly to S3 cloud storage. |
| **expo-image-picker** | 17 | Opens the camera or photo library so the participant can choose a file. |
| **expo-video** | 3 | Plays back video answers in the app. |
| **expo-device** | 55 | Detects if running on a real device (push tokens only work on real devices). |

---

## Folder Structure — What Does Each File Do?

```
MobileEtnoV2_frontendapp/
│
├── index.ts                    ← Entry point. Registers App.tsx with Expo.
├── App.tsx                     ← Root component. Sets up push notifications and returns AppNavigator.
│
├── src/
│   │
│   ├── config.ts               ← Reads the API base URL from the environment variable.
│   │
│   ├── navigation/
│   │   └── AppNavigator.tsx    ← Defines all screens and their order/transitions.
│   │
│   ├── screens/                ← One file = one screen the user sees.
│   │   ├── LoginScreen.tsx             ← Email + password form.
│   │   ├── ChangePasswordScreen.tsx    ← Shown on first login to set a permanent password.
│   │   ├── ProjectsScreen.tsx          ← Lists all projects assigned to this participant.
│   │   ├── ProjectDetailScreen.tsx     ← Lists tasks inside a project.
│   │   ├── TaskDetailScreen.tsx        ← Shows questions in a task. Participant answers here.
│   │   ├── ChatScreen.tsx              ← Real-time chat with a moderator about a question.
│   │   └── NotificationsScreen.tsx     ← Lists all messages from moderators.
│   │
│   ├── services/               ← Functions that call the backend API.
│   │   ├── authService.ts      ← Login, change password.
│   │   ├── projectService.ts   ← Get projects.
│   │   ├── taskService.ts      ← Get tasks.
│   │   ├── questionService.ts  ← Get questions.
│   │   ├── attachmentService.ts ← File upload (init → S3 → confirm) and cover images.
│   │   ├── messageService.ts   ← Chat history, WebSocket URL builder, notifications.
│   │   └── pushTokenService.ts ← Registers/unregisters Expo push token with backend.
│   │
│   ├── types/                  ← TypeScript types (shapes of data from the API).
│   │   ├── auth.ts             ← LoginRequest, LoginResponse, etc.
│   │   ├── project.ts          ← Project type.
│   │   └── task.ts             ← Task type.
│   │
│   └── utils/
│       └── storage.ts          ← Wrappers around AsyncStorage (save/load the JWT token).
│
├── assets/                     ← App icons and splash screen images.
│   ├── icon.png                ← App icon (used on home screen).
│   ├── adaptive-icon.png       ← Android adaptive icon.
│   ├── splash-icon.png         ← Image shown while the app is loading.
│   └── favicon.png             ← Used if running in a web browser.
│
├── eas.json                    ← EAS Build configuration (how to build for dev/preview/production).
├── package.json                ← Dependencies and npm scripts.
├── tsconfig.json               ← TypeScript compiler settings.
└── .env.example                ← Template showing which environment variables are needed.
```

---

## All Screens — What Each Screen Does

### Screen Flow

```
LoginScreen
    │
    ├─ (first login) → ChangePasswordScreen → LoginScreen
    │
    └─ (normal login) → ProjectsScreen
                              │
                              └─ ProjectDetailScreen (one project)
                                        │
                                        └─ TaskDetailScreen (one task, answer questions)
                                                  │
                                                  └─ ChatScreen (chat about one question)

From the header:  → NotificationsScreen (all unread messages)
```

### Screen Descriptions

| Screen | File | What it shows |
|---|---|---|
| **Login** | `LoginScreen.tsx` | Email + password fields. Calls `POST /api/auth/app/login`. |
| **Change Password** | `ChangePasswordScreen.tsx` | Required on first login. Gets a temp token from login, calls `POST /api/auth/change-initial-password`. |
| **Projects** | `ProjectsScreen.tsx` | A list of all projects this participant is enrolled in. Shows cover images. Tapping navigates to ProjectDetail. |
| **Project Detail** | `ProjectDetailScreen.tsx` | Lists tasks inside a project, sorted by `sort_order`. Shows cover images and a "✓ Done" badge on completed tasks. |
| **Task Detail** | `TaskDetailScreen.tsx` | Shows all questions in a task. The participant types/records their answers here. Each question type has its own input. After all required questions are answered, the task is marked done locally. |
| **Chat** | `ChatScreen.tsx` | Real-time WebSocket chat between the participant and moderators, specific to one question. Shows a "Connected" / "Disconnected" indicator. |
| **Notifications** | `NotificationsScreen.tsx` | Lists all conversations where a moderator has sent an unread message. Tapping opens the ChatScreen. |

---

## Navigation Structure

Navigation is defined in `src/navigation/AppNavigator.tsx`.

It uses a **Native Stack** (feels native on both iOS and Android):

```typescript
type RootStackParamList = {
    Login: undefined                              // no params
    ChangePassword: { token: string }             // receives a temp JWT
    Projects: undefined                           // no params
    Notifications: undefined                      // no params
    ProjectDetail: { projectId: string; projectName: string }
    TaskDetail: { projectId: string; taskId: string; taskTitle: string }
    Chat: {
        projectId: string
        participantId: string
        questionId: string
        questionDescription: string
    }
}
```

The header bar is styled with a purple background (`#7C3AED`) and white text across all screens.

---

## How Authentication Works on the App

### Logging In

1. User enters email and password.
2. App calls `POST /api/auth/app/login`.
3. If the account requires a password change → server returns a `temp_token` → app navigates to `ChangePasswordScreen`.
4. If login succeeds → server returns a `full_token` → app saves it in AsyncStorage and navigates to `Projects`.

### Staying Logged In

The JWT token is stored with AsyncStorage (the phone's local storage):

```typescript
// utils/storage.ts
export const setFullToken = (token: string) => AsyncStorage.setItem('full_token', token)
export const getFullToken = () => AsyncStorage.getItem('full_token')
export const clearTokens = () => AsyncStorage.removeItem('full_token')
```

Unlike the web app (which uses sessionStorage that clears on tab close), AsyncStorage **persists between app restarts**. If you close and reopen the app, you're still logged in.

### Sending the Token with API Calls

Every service function reads the token from AsyncStorage and puts it in the Authorization header:

```typescript
const authHeader = async () => ({
    headers: { Authorization: `Bearer ${await getFullToken()}` }
})
```

### Logging Out

Calling `clearTokens()` removes the token. The app then navigates back to the Login screen. There is no explicit logout button shown in the code — it can be added by calling `clearTokens()` and navigating to `'Login'`.

---

## How the App Connects to the Backend

The backend URL is configured in **one place**:

```typescript
// src/config.ts
export const API_BASE_URL = process.env.EXPO_PUBLIC_API_BASE_URL ?? 'http://localhost:3000';
```

And set in `.env` (which you must create from `.env.example`):

```env
EXPO_PUBLIC_API_BASE_URL=http://YOUR_LOCAL_IP:3000
```

**Important:** During development you cannot use `localhost` from a phone or emulator — the phone doesn't know what "localhost" means. You must use your computer's local network IP address (e.g., `192.168.1.10`).

For production, this will be your live server URL (e.g., `https://api.yourcompany.com`).

---

## Real-Time Chat (WebSocket)

The chat screen uses the browser's native `WebSocket` API (no extra library needed).

### How it works:

```
1. Load chat history (HTTP GET):
   GET /api/projects/{p}/participants/{u}/questions/{q}/history

2. Open WebSocket connection:
   wss://api.example.com/api/ws/projects/{p}/participants/{u}/questions/{q}/get?token=<jwt>

3. Send a message:
   ws.send(JSON.stringify({ content: "Hello!" }))

4. Receive a message:
   ws.onmessage = (event) => {
       const msg = JSON.parse(event.data)
       // add msg to the messages list
   }

5. On component unmount (leaving the screen):
   ws.close()
```

The JWT token is passed as a **query parameter** in the WebSocket URL (not a header — browsers don't support custom headers in WebSocket connections).

---

## Push Notifications

The app uses **Expo Push Notifications**.

### How they work:

1. When a logged-in user opens the app, `setupPushNotifications()` runs in `App.tsx`.
2. It asks the user for permission to send notifications.
3. On Android, it creates a notification channel called "default".
4. It calls Expo's SDK to get an **Expo Push Token** (a unique identifier for this device).
5. It registers that token with the backend: `POST /api/push-tokens`.

When a moderator sends a chat message:
1. The backend calls Expo's push service with the participant's push token.
2. Expo delivers the notification to the device.
3. The notification shows: project name, question description, message preview.
4. Tapping the notification navigates the app directly to the `ChatScreen` for that conversation.

### Note on Simulators

Push tokens do not work on iOS Simulator or Android Emulator — only on real physical devices. The code silently ignores errors from simulators.

---

## File Upload Flow

When a participant uploads a photo or video answer, the app uses a 3-step process to send large files directly to S3 (cloud storage) without routing them through the backend server:

```
Step 1: Tell the backend you're about to upload
        → POST /api/projects/{p}/tasks/{t}/questions/{q}/answers/{a}/attachments/init
        ← Returns: { attachment_id, upload_url, file_key }

Step 2: Upload the file directly to S3
        → expo-file-system.uploadAsync(upload_url, fileUri, { httpMethod: 'PUT' })
        ← S3 returns 200 OK

Step 3: Tell the backend the upload is done
        → POST /api/.../attachments/{attachment_id}/confirm
        ← Backend links the file to the answer
```

This pattern keeps the backend fast and simple — it never touches the file bytes.

---

## Local Storage — What Gets Saved on the Phone

```typescript
// Keys stored in AsyncStorage:
'full_token'       // The JWT login token
'completed_tasks'  // Array of task IDs the user has completed (JSON string)
```

`completed_tasks` is stored locally (not in the backend). This is used to show the "✓ Done" badge on the Project Detail screen. If the user logs in on a different device, this badge won't appear until they complete the task again on that device.

---

## Environment Variables

Only one environment variable is needed:

```env
# .env  (create this file in the root of the project)
EXPO_PUBLIC_API_BASE_URL=http://192.168.1.10:3000
```

Variables prefixed with `EXPO_PUBLIC_` are automatically included in the app bundle by Expo. They are visible in the compiled app — never put secrets here.

For production builds, this should be your live API URL:
```env
EXPO_PUBLIC_API_BASE_URL=https://api.yourcompany.com
```

---

## How to Run Locally (Development)

### Prerequisites

- Node.js 18+
- Expo CLI: `npm install -g expo-cli` (or use `npx expo`)
- Expo Go app on your phone (free, from App Store / Google Play)
- OR: Android Studio (for Android Emulator) / Xcode (for iOS Simulator on Mac)

### Steps

```bash
# Step 1: Go into the project folder (important: use C:\Projects, not OneDrive)
cd C:\Projects\MobileEtnoV2_frontendapp

# Step 2: Install dependencies
npm install

# Step 3: Create the .env file
# Copy .env.example to .env and set your computer's local IP address
# Example:
echo "EXPO_PUBLIC_API_BASE_URL=http://192.168.1.10:3000" > .env

# Step 4: Make sure the backend is running

# Step 5: Start Expo
npm run start
# or: npx expo start
```

Expo will show a **QR code** in the terminal.

- **On your phone:** Open the Expo Go app and scan the QR code.
- **On Android emulator:** Press `a` in the terminal.
- **On iOS simulator (Mac only):** Press `i` in the terminal.

### Other Commands

```bash
npm run android    # Start on Android emulator directly
npm run ios        # Start on iOS simulator directly (Mac only)
npm run web        # Start in a web browser (limited native features)
```

---

## Building for Production (EAS Build)

The app is built and distributed using **EAS Build** (Expo Application Services). This is a cloud build service — you don't need Xcode or Android Studio installed locally to produce the final `.ipa` (iOS) or `.apk`/`.aab` (Android) files.

### EAS Build Profiles (`eas.json`)

```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {
      "autoIncrement": true
    }
  }
}
```

| Profile | Who it's for | What it produces |
|---|---|---|
| `development` | Developers | A debug build you can install internally and use with Expo Dev Client. |
| `preview` | QA testers | A release build distributed internally (not via App Store/Play Store). |
| `production` | End users | A production build submitted to App Store / Google Play. Version number auto-increments. |

### Build Commands

```bash
# Install EAS CLI (once)
npm install -g eas-cli

# Log in to your Expo account
eas login

# Build for Android (production)
eas build --platform android --profile production

# Build for iOS (production)
eas build --platform ios --profile production

# Build for both at once
eas build --platform all --profile production
```

### Before Your First Production Build

1. You need an **Expo account** (free at expo.dev).
2. You need an **EAS project** linked to this app. Run `eas init` to create one.
3. The EAS Project ID must be added to your `app.json` under `expo.extra.eas.projectId`. This is what push notifications use.
4. For iOS: you need an Apple Developer account ($99/year) to submit to the App Store.
5. For Android: you need a Google Play Developer account ($25 one-time) to submit to the Play Store.

### Submitting to App Stores

```bash
# Submit the latest iOS build to App Store Connect
eas submit --platform ios --latest

# Submit the latest Android build to Google Play
eas submit --platform android --latest
```

---

## Deployment Checklist

### For Development (Local Testing)

- [ ] `node_modules` installed (`npm install`)
- [ ] `.env` file created with correct local IP
- [ ] Backend running on the same network
- [ ] Expo Go installed on test device

### For Production Build

- [ ] `.env` updated with production API URL (`EXPO_PUBLIC_API_BASE_URL=https://api.yourcompany.com`)
- [ ] `app.json` has the correct EAS Project ID in `expo.extra.eas.projectId`
- [ ] EAS CLI installed and logged in
- [ ] `eas build --platform all --profile production` run successfully
- [ ] Build tested on a real device before submitting

### For App Store / Play Store Submission

**iOS:**
- [ ] Apple Developer Program membership active
- [ ] App bundle ID configured in Expo
- [ ] App Store Connect listing created (name, description, screenshots)
- [ ] Privacy policy URL provided
- [ ] `eas submit --platform ios` run

**Android:**
- [ ] Google Play Console account active
- [ ] App created in Play Console
- [ ] `eas submit --platform android` run

---

## Common Issues

| Problem | Cause | Fix |
|---|---|---|
| Can't connect to backend on phone | Using `localhost` in API URL | Use your computer's local IP (e.g., `192.168.1.10`) |
| Push notifications not working | Running on a simulator | Test on a real device. Silently fails on simulators by design. |
| Build fails with "Permission denied" | Running from OneDrive folder | Use `C:\Projects\MobileEtnoV2_frontendapp` — NOT the OneDrive copy |
| "No tasks" showing after completing a task | `completed_tasks` cleared | `AsyncStorage` persists across sessions; if it's cleared the badge disappears |
| EAS build can't find project ID | Missing `eas.projectId` in app.json | Run `eas init` and it will add it automatically |
