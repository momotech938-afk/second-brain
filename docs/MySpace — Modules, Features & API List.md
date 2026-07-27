# MySpace — Modules, Features & API List

Oct 29, 2026 · @Pranjal Goyal

Every module, its features, and every REST endpoint the Angular app needs from the NestJS API. Modules marked v1 ship first; the rest follow in v1.1.

## 1. API conventions

All endpoints live under `/api/v1`, need a JWT access token unless marked Public, and return the same list and error shapes.

| Rule | Value |
| --- | --- |
| Base URL | `/api/v1` |
| Auth header | `Authorization: Bearer <accessToken>` (15 min) |
| Refresh token | httpOnly cookie `rt`, path `/api/v1/auth` |
| IDs in URLs | `:id` = 26-char ULID (public\_id), never the numeric DB id |
| List query params | `page`, `limit` (max 100), `q` (search), `sort` (e.g. `-updatedAt`), module filters |
| List response | `{ data: [...], meta: { total, page, limit } }` |
| Error response | `{ statusCode, message, path, requestId, timestamp }` |
| Delete | soft delete (moves to Trash); `204 No Content` |
| Other user's data | `404 Not Found` |
| Docs | Swagger at `/api/docs` (dev only) |

Every content module follows the same 6 CRUD endpoints, written once here and listed per module below as **CRUD**:

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/{module}` | list with filters + pagination |
| GET | `/{module}/:id` | get one |
| POST | `/{module}` | create |
| PATCH | `/{module}/:id` | update (partial) |
| DELETE | `/{module}/:id` | soft delete to Trash |
| POST | `/{module}/:id/restore` | restore from Trash |

## 2. Auth, Profile & Settings (v1)

### 2.1 Auth

**Features:** email + password login, Google login, logout, refresh session, forgot/reset password, optional 2FA (TOTP), re-auth before sensitive actions, active sessions list, auto-lock on inactivity (frontend).

| Method | Endpoint | Purpose | Access |
| --- | --- | --- | --- |
| POST | `/auth/register` | first-time owner setup (disabled after first user) | Public |
| POST | `/auth/login` | email + password login | Public |
| GET | `/auth/google` | start Google OAuth | Public |
| GET | `/auth/google/callback` | Google returns here, issue tokens | Public |
| POST | `/auth/refresh` | new access token from cookie, rotate refresh | Public (cookie) |
| POST | `/auth/logout` | revoke current refresh token | Auth |
| POST | `/auth/logout-all` | revoke all sessions | Auth |
| POST | `/auth/forgot-password` | email a reset link | Public |
| POST | `/auth/reset-password` | set new password with token | Public |
| POST | `/auth/change-password` | change password when logged in | Auth |
| POST | `/auth/reauth` | confirm password/TOTP before reveal/export | Auth |
| GET | `/auth/sessions` | list active devices | Auth |
| DELETE | `/auth/sessions/:id` | sign out one device | Auth |
| POST | `/auth/2fa/setup` | generate secret + QR code | Auth |
| POST | `/auth/2fa/enable` | verify first code, turn on | Auth |
| POST | `/auth/2fa/verify` | second step of login | Temp token |
| POST | `/auth/2fa/disable` | turn off (needs code) | Auth |

### 2.2 Profile

**Features:** view/edit name, email, bio, avatar; link/unlink Google.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/me` | current user profile |
| PATCH | `/me` | update name, bio |
| POST | `/me/avatar` | upload avatar (multipart, max 2 MB) |
| DELETE | `/me/avatar` | remove avatar |
| DELETE | `/me/google` | unlink Google account |

### 2.3 Settings

**Features:** theme (light/dark/system), language, default view (sidebar), offline mode toggle, auto backup toggle, lock timeout, notification preferences.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/settings` | user settings |
| PATCH | `/settings` | update any setting |
| GET | `/bootstrap` | one call on app load: user, settings, theme, nav, widgets, quick actions, feature flags |

## 3. Dashboard, Notes & Credentials (v1)

### 3.1 Dashboard

**Features:** greeting with name, date/time and quote; quick action cards (New Note, Add Credential, Add Memory, Upload File); quick overview counts (Notes, Credentials, Memories, Tasks, Files, Events, Favorites, Trash); recent activity; today's tasks; this week's progress bar. Layout comes from admin widget config.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/dashboard/summary` | all overview counts in one call |
| GET | `/dashboard/today` | today's tasks + today's events |
| GET | `/dashboard/week-progress` | done / total tasks this week |
| GET | `/quotes/random` | greeting quote |

Recent activity uses `GET /activity` (section 6). Widget layout comes from `/bootstrap`.

### 3.2 Notes

**Features:** rich-text notes (Tiptap), title, colour, category, tags; grid view; category chips (All, Work, Personal, Learning, Ideas, Travel, Finance); search notes; pin; favorite; duplicate; soft delete + restore; sort by updated/created/title.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| CRUD | `/notes` | list (`category`, `tag`, `q`, `favorite`, `pinned`, `sort`), get, create, update, delete, restore |
| PATCH | `/notes/:id/pin` | pin / unpin |
| POST | `/notes/:id/duplicate` | copy a note |

### 3.3 Credentials Vault

**Features:** store login (name, URL, username, password, notes, icon, category); categories (Work, Personal, Finance, Social, Other); search by name/URL; password hidden by default; reveal with re-auth; one-click copy (auto-clears clipboard after 30 s); password generator; last-viewed time; audit log of every reveal; secrets encrypted at rest (AES-256-GCM).

| Method | Endpoint | Purpose |
| --- | --- | --- |
| CRUD | `/credentials` | list (`category`, `q`) returns no secrets; get, create, update, delete, restore |
| POST | `/credentials/:id/reveal` | return decrypted password; needs recent `/auth/reauth`; rate-limited; audited |
| POST | `/credentials/:id/copy-event` | log that the password was copied (audit) |
| GET | `/credentials/generate-password` | random strong password (`length`, `symbols`, `numbers`) |
| GET | `/credentials/:id/history` | previous passwords (encrypted), optional v1.1 |

## 4. Tasks, Goals, Habits & Calendar

### 4.1 Tasks (v1)

**Features:** add task with title, notes, due date/time, priority, category (Work, Learning, Personal, Side Project); grouped as Today / Tomorrow / This Week / Upcoming / Completed; check off; repeat tasks; link to a goal; reorder.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| CRUD | `/tasks` | list (`view=today\|tomorrow\|week\|upcoming\|completed`, `category`, `goalId`), get, create, update, delete, restore |
| PATCH | `/tasks/:id/toggle` | mark done / undone |
| PATCH | `/tasks/reorder` | save new order `[{ id, sortOrder }]` |
| GET | `/tasks/stats` | counts per view + week progress |

### 4.2 Goals (v1)

**Features:** goal with title, target date, status; progress % from linked tasks.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| CRUD | `/goals` | list, get, create, update, delete, restore |
| GET | `/goals/:id/tasks` | tasks linked to a goal |

### 4.3 Habits (v1.1)

**Features:** habit with name and frequency (daily / chosen days); tick off per day; streak count; monthly heatmap.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| CRUD | `/habits` | list, get, create, update, delete, restore |
| POST | `/habits/:id/logs` | mark a date done `{ date }` |
| DELETE | `/habits/:id/logs/:date` | unmark a date |
| GET | `/habits/:id/stats` | current streak, best streak, completion % (`from`, `to`) |

### 4.4 Calendar & Events (v1.1)

**Features:** month / week / day views; events with title, start/end, all-day, location, colour, category (Work, Personal); repeat (birthdays); reminders; tasks with due dates shown on the calendar; upcoming events list.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| CRUD | `/events` | list, get, create, update, delete, restore |
| GET | `/calendar` | events + due tasks in a range (`from`, `to`), repeats expanded |
| GET | `/events/upcoming` | next N events (`limit`) |
| PATCH | `/events/:id/move` | drag-drop to new date/time |

## 5. Memories & Files (v1.1)

### 5.1 Memories & Journal

**Features:** memory with title, description, date, location (name + lat/lng), mood, category (Trip, Office Life, Friends, Personal), multiple photos; Timeline view grouped by month; Gallery view of all photos; Map view with pins; search; favorite.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| CRUD | `/memories` | list (`from`, `to`, `category`, `q`), get, create, update, delete, restore |
| GET | `/memories/timeline` | memories grouped by month (`year`) |
| GET | `/memories/gallery` | all photos across memories, paginated |
| GET | `/memories/map` | memories with lat/lng for map pins |
| POST | `/memories/:id/media` | attach uploaded files `{ fileIds: [] }` |
| DELETE | `/memories/:id/media/:fileId` | detach a photo |
| PATCH | `/memories/:id/media/reorder` | change photo order |

### 5.2 Files & Documents

**Features:** folders (nested) with colour; upload any file (PDF, images, DOCX, XLSX, ZIP, code) up to 50 MB; filter tabs (All, Documents, Images, PDF, Code, Others); file size + date; preview images and PDFs; download; rename; move between folders; thumbnails for images; storage used.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| CRUD | `/folders` | list (`parentId`), get, create, update (rename/colour), delete, restore |
| GET | `/folders/:id/contents` | sub-folders + files in one call |
| POST | `/files/upload-url` | presigned S3 upload URL `{ name, type, size }` |
| POST | `/files/complete` | save file metadata after S3 upload |
| GET | `/files` | list (`folderId`, `type=document\|image\|pdf\|code\|other`, `q`) |
| GET | `/files/:id` | file metadata |
| PATCH | `/files/:id` | rename or move `{ name, folderId }` |
| GET | `/files/:id/download-url` | presigned download link (5 min) |
| GET | `/files/:id/preview-url` | presigned inline view link |
| DELETE | `/files/:id` | soft delete to Trash |
| POST | `/files/:id/restore` | restore |
| GET | `/files/storage` | total bytes used, by type |

**Local dev note:** until AWS is set up, the same endpoints can save to a local `uploads/` folder behind a StorageService interface; switch to S3 later without changing the API.

## 6. Shared modules (v1)

These serve every module, so they are built once and reused.

### 6.1 Categories

**Features:** per-module categories with name, colour, icon and order (the filter chips on every screen).

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/categories` | list (`module=note\|credential\|task\|memory\|file\|event`) |
| POST | `/categories` | create |
| PATCH | `/categories/:id` | rename, recolour |
| DELETE | `/categories/:id` | delete (items become uncategorised) |
| PATCH | `/categories/reorder` | save chip order |

### 6.2 Tags

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/tags` | list all tags (`q` for autocomplete) |
| POST | `/tags` | create |
| PATCH | `/tags/:id` | rename |
| DELETE | `/tags/:id` | delete and remove from items |

### 6.3 Global search

**Features:** top-bar search across notes, credentials (name/URL only), tasks, memories, files, events; grouped results; Ctrl+K shortcut.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/search` | `q`, optional `types=note,task,...`, `limit` per type |

### 6.4 Favorites

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/favorites` | all favorited items (`type` filter) |
| POST | `/favorites` | add `{ entityType, entityId }` |
| DELETE | `/favorites/:entityType/:entityId` | remove |

### 6.5 Trash

**Features:** everything deleted in one place; restore; delete forever; empty trash; auto-purge after 30 days.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/trash` | all soft-deleted items (`type` filter), with days left |
| POST | `/trash/:entityType/:entityId/restore` | restore one item |
| DELETE | `/trash/:entityType/:entityId` | delete forever (needs confirm) |
| DELETE | `/trash` | empty trash (needs reauth) |

### 6.6 Recent activity

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/activity` | recent actions (`limit`, `type`), feeds dashboard |

### 6.7 Notifications & reminders (v1.1)

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/notifications` | list (`unread=true`) |
| GET | `/notifications/unread-count` | bell badge number |
| PATCH | `/notifications/:id/read` | mark one read |
| PATCH | `/notifications/read-all` | mark all read |

## 7. Admin panel (v1)

All endpoints under `/admin/*` need role `admin` or `super_admin`. Every save bumps `settings_version` so the app reloads `/bootstrap`.

| Admin module | Features | Method | Endpoint |
| --- | --- | --- | --- |
| Admin dashboard | storage used, item counts, last logins, recent admin changes | GET | `/admin/overview` |
| Branding | app name, logo, favicon, greeting text | GET | `/admin/branding` |
|  |  | PATCH | `/admin/branding` |
|  |  | POST | `/admin/branding/logo` |
| Quotes | add/edit/remove greeting quotes | CRUD | `/admin/quotes` |
| Theme builder | primary/accent/surface colours, radius, font, light + dark tokens, live preview, set default | GET | `/admin/themes` |
|  |  | POST | `/admin/themes` |
|  |  | PATCH | `/admin/themes/:id` |
|  |  | PATCH | `/admin/themes/:id/default` |
|  |  | DELETE | `/admin/themes/:id` |
| Navigation manager | add, rename, hide, reorder (drag), pick icon for sidebar items | GET | `/admin/nav` |
|  |  | POST | `/admin/nav` |
|  |  | PATCH | `/admin/nav/:id` |
|  |  | PATCH | `/admin/nav/reorder` |
|  |  | DELETE | `/admin/nav/:id` |
| Dashboard widget builder | add/remove widgets, size, order, config per widget | GET | `/admin/widgets` |
|  |  | GET | `/admin/widgets/types` |
|  |  | POST | `/admin/widgets` |
|  |  | PATCH | `/admin/widgets/:id` |
|  |  | PATCH | `/admin/widgets/reorder` |
|  |  | DELETE | `/admin/widgets/:id` |
| Quick actions | the 4 dashboard action cards: label, icon, colour, route | CRUD | `/admin/quick-actions` |
|  |  | PATCH | `/admin/quick-actions/reorder` |
| Feature flags | turn modules on/off (Memories, Files, Calendar, Habits…) | GET | `/admin/flags` |
|  |  | PATCH | `/admin/flags/:key` |
| Security policy | lock timeout, 2FA required, session length, reveal needs reauth | GET | `/admin/security-policy` |
|  |  | PATCH | `/admin/security-policy` |
| Users | list users, change role, disable (ready for multi-user later) | GET | `/admin/users` |
|  |  | PATCH | `/admin/users/:id` |
| Audit logs | logins, failed logins, reveals, exports, admin changes; filter by action/date | GET | `/admin/audit-logs` |
| Backup & export | export all data (encrypted JSON + files zip), import/restore, backup history | POST | `/admin/backup/export` |
|  |  | GET | `/admin/backup/:jobId` |
|  |  | GET | `/admin/backup/:jobId/download` |
|  |  | POST | `/admin/backup/import` |
|  |  | GET | `/admin/backup/history` |

## 8. System endpoints & totals

| Method | Endpoint | Purpose | Access |
| --- | --- | --- | --- |
| GET | `/health` | DB ping, used by load balancer | Public |
| GET | `/version` | API version + build time | Public |

**179 endpoints in total: 128 for v1, 51 for v1.1** (each CRUD counted as 6).

| Area | Release | Endpoints |
| --- | --- | --- |
| Admin panel | v1 | 45 |
| Auth, Profile, Settings | v1 | 25 |
| Shared (categories, tags, search, favorites, trash, activity) | v1 | 18 |
| Notes | v1 | 8 |
| Credentials | v1 | 10 |
| Tasks + Goals | v1 | 16 |
| Dashboard | v1 | 4 |
| System | v1 | 2 |
| Files + Folders | v1.1 | 17 |
| Memories | v1.1 | 12 |
| Habits | v1.1 | 9 |
| Calendar & Events | v1.1 | 9 |
| Notifications | v1.1 | 4 |

Build order follows the roadmap: Auth → Settings/Bootstrap → Shared → Notes → Tasks/Goals → Dashboard → Credentials → Admin → v1.1 modules.
