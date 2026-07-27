# Myspce Project Context

This file separates what is **confirmed** about Myspce from what this guide **assumes** or **proposes**. Read it before the modules, and update it when your decisions change.

Back to [README](README.md).

> **Name note:** the app's UI says "MySpace" (see the dashboard design). Your planning documents call the project "Myspce". This guide uses **Myspce** for the project and "MySpace" only where the UI label is quoted.

---

## 1. What was inspected

| Source | What it gave us |
|---|---|
| Design screenshot (8 screens) | Modules, layout, navigation, components, light/dark themes |
| "MySpace — Personal Vault: Build Plan & Architecture" doc | Stack, schema proposal, admin-panel design, security plan |
| "MySpace — Modules, Features & API List" doc | Module list and a **planned** REST API (179 endpoints) |
| "MySpace — Modules & Roadmap" doc | 41 modules, 9-phase roadmap |
| Your decisions in chat (29 Oct 2026) | Single user, auth methods, styling, hosting, security level |

**No source code exists yet.** There is no frontend or backend repository to inspect. Every API in this guide is from the planned contract, not a running server.

## 2. Confirmed facts

### Product purpose
A **personal vault website** for one person to store and organise notes, credentials, memories, tasks and goals, files and calendar events, with an **admin panel** that customises the app (theme, navigation, dashboard widgets, categories) without code changes.

### Confirmed decisions

| Topic | Decision |
|---|---|
| Users | **Single user** (the owner). Tables still carry `user_id` so multi-user is possible later. |
| Login | **Google login + email/password** |
| Vault security | **Server-side encryption** (AES-256-GCM) now; zero-knowledge later |
| Styling | **Tailwind CSS** |
| Devices | **Mainly mobile-responsive web** (no PWA/offline in v1) |
| Hosting | Local development first, **free-tier AWS** later |
| Frontend | **Angular** |
| Backend | **NestJS + TypeORM + MySQL** (planned, not built) |

### Confirmed screens (from the design)

| # | Screen | What it shows |
|---|---|---|
| 1 | Dashboard (dark) | Greeting with name, date/time, quote; 4 quick actions (New Note, Add Credential, Add Memory, Upload File); overview counts (Notes 124, Credentials 28, Memories 48, Tasks 12, Files 68, Events 6, Favorites 23, Trash 4); Recent Activity; Today's Tasks; "This Week Progress" bar |
| 2 | Notes (light) | Search, category chips (All, Work, Personal, Learning, Ideas, Travel, Finance), colourful note cards with title, preview, date, category tag, "New Note" |
| 3 | Credentials (dark) | Search, chips (All, Work, Personal, Finance, Social, Other), rows with service icon, name, URL, username, masked password, copy button, category tag, "Add Credential" |
| 4 | Memories (light) | Timeline / Gallery / Map tabs, month groups, memory with title, location, photos, description, "Add Memory" |
| 5 | Tasks & Goals (light) | Tabs Tasks / Goals / Habits, filters All / Today / Upcoming / Completed, groups Today / Tomorrow / This Week, checkbox, category tag, time |
| 6 | Files (dark) | Tabs All / Documents / Images / PDF / Code / Others, folders and files with size and date, "Upload" |
| 7 | Calendar (light) | Month grid, Month / Week / Day, coloured events, Upcoming Events list, "Add Event" |
| 8 | Settings (dark) | Tabs Profile / Appearance / Security / Customization / Backup; name, email, bio, photo; theme Light / Dark / System; language; default view; toggles for offline mode, auto backup, lock on inactivity |

### Confirmed layout
- Left **sidebar**: logo "MySpace", user avatar + name + email, nav items (Dashboard, Notes, Credentials, Memories, Tasks & Goals, Files, Calendar, Favorites, Trash), Settings at the bottom.
- **Top bar**: global search ("Search anything…"), theme toggle, notifications bell, avatar.
- Both **light and dark** themes.

### v1 scope (your pick)
Dashboard, Notes, Credentials, Tasks (+ Goals), Settings, Admin. Memories, Files, Calendar and Habits are **v1.1**.

## 3. Planned API (from the API list doc)

The backend is not built. These endpoints are the **agreed plan**. Examples in this guide use them; if the real backend differs later, update this section and the affected notes.

| Convention | Value |
|---|---|
| Base URL | `/api/v1` |
| Auth | `Authorization: Bearer <accessToken>` (15 min); refresh token in an httpOnly cookie |
| IDs | 26-char ULID strings |
| List response | `{ data: T[], meta: { total, page, limit } }` |
| Error response | `{ statusCode, message, path, requestId, timestamp }` |
| Delete | soft delete → Trash |

Endpoints used most in this guide:

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/login`, `GET /auth/google`, `POST /auth/refresh`, `POST /auth/logout`, `POST /auth/reauth` |
| Bootstrap | `GET /bootstrap` (user, settings, theme, nav, widgets, flags) |
| Dashboard | `GET /dashboard/summary`, `GET /dashboard/today`, `GET /dashboard/week-progress` |
| Notes | `GET/POST /notes`, `GET/PATCH/DELETE /notes/:id`, `POST /notes/:id/restore`, `PATCH /notes/:id/pin` |
| Credentials | CRUD `/credentials` (list never returns secrets), `POST /credentials/:id/reveal` |
| Tasks | CRUD `/tasks?view=today|tomorrow|week|upcoming|completed`, `PATCH /tasks/:id/toggle` |
| Categories | `GET /categories?module=note|credential|task|…` |
| Search | `GET /search?q=` |
| Files | `POST /files/upload-url`, `POST /files/complete`, `GET /files/:id/download-url` |

## 4. Data entities used in examples

These TypeScript interfaces mirror the planned schema. Field names are **proposed**; adjust when the backend exists.

```typescript
// src/app/core/models/index.ts  (proposed)
export type Ulid = string;

export interface Paginated<T> {
  data: T[];
  meta: { total: number; page: number; limit: number };
}

export interface User {
  id: Ulid;
  name: string;
  email: string;
  avatarUrl: string | null;
  bio: string | null;
  role: 'super_admin' | 'admin' | 'user';
}

export type CategoryModule = 'note' | 'credential' | 'task' | 'memory' | 'file' | 'event';

export interface Category {
  id: Ulid;
  module: CategoryModule;
  name: string;
  slug: string;
  color: string;
}

export interface Note {
  id: Ulid;
  title: string;
  preview: string;          // first ~120 chars, computed by the API
  color: string | null;
  pinned: boolean;
  isFavorite: boolean;
  category: Category | null;
  tags: string[];
  createdAt: string;        // ISO date
  updatedAt: string;
}

export interface CredentialSummary {   // list view: never contains the secret
  id: Ulid;
  name: string;
  url: string | null;
  username: string;
  icon: string | null;
  category: Category | null;
  lastViewedAt: string | null;
}

export interface Task {
  id: Ulid;
  title: string;
  notes: string | null;
  dueAt: string | null;
  priority: 'low' | 'medium' | 'high';
  status: 'todo' | 'done';
  category: Category | null;
  goalId: Ulid | null;
}
```

## 5. Assumptions (not confirmed)

| # | Assumption | Why it's assumed |
|---|---|---|
| A1 | The app is a client-rendered SPA behind login (no SSR in v1) | Content is private, so SEO doesn't matter |
| A2 | Access token kept in memory; refresh token in httpOnly cookie | Recommended in the build plan; final backend not built |
| A3 | Admin role exists even with a single user | The admin panel is confirmed; RBAC kept simple |
| A4 | Icons from Lucide | The designs use a single outline icon style |
| A5 | Notes use a rich-text editor (Tiptap) storing JSON | Proposed in the build plan |
| A6 | A mock backend (JSON server or Angular in-memory interceptor) is used during learning | No backend exists yet |

## 6. Proposed frontend architecture

```text
src/app/
  app.ts / app.html / app.config.ts / app.routes.ts
  core/
    auth/          auth-store.ts, auth-interceptor.ts, auth-guard.ts
    http/          api-url.token.ts, error-interceptor.ts
    layout/        shell.ts, sidebar.ts, topbar.ts
    theme/         theme-store.ts
    bootstrap/     bootstrap-store.ts
    models/        index.ts
  shared/
    ui/            card, chip-filter, page-header, empty-state, modal, toast, toggle, search-box, icon
    directives/    copy-to-clipboard, has-role, autofocus
    pipes/         file-size, time-ago
  features/
    dashboard/  notes/  credentials/  tasks/  settings/   (v1)
    memories/   files/  calendar/                          (v1.1)
      data-access/   *-api.ts, *-store.ts
      pages/         route components
      components/    feature-only UI
  admin/           branding, theme, navigation, widgets, flags, audit-log
```

Rules: features never import from each other; shared code goes to `shared/` or `core/`.

## 7. Unresolved questions

1. Final API shapes (field names, error codes) once the NestJS backend exists.
2. Whether the admin panel lives inside the same Angular app (assumed) or a separate app.
3. 2FA: optional or skipped in v1? (Assumed optional.)
4. Domain name and the exact AWS hosting (S3 + CloudFront vs EC2 + nginx).
5. Rich-text editor choice for Notes (Tiptap assumed).
6. Design tokens: final colours and fonts are not yet exported from Figma.
7. Note body field: the planned `Note` has `preview` only. Labs assume `GET /notes/:id` adds `content` (`NoteDetail`). Confirm the name and format (Tiptap JSON vs HTML).
8. Category counts for chips: not in the planned API. Labs show chips without counts until decided.
