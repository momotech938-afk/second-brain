# Implementation Plan — The Real Myspce Frontend

Back to [Module 22](README.md) · Previous: [Debugging exercises](03-debugging-exercises.md) · Next: [Self-assessment rubric](05-self-assessment-rubric.md)

This is the plan for building the production Myspce frontend **after** you pass the rubric. It reuses the decisions in [MYSPCE_PROJECT_CONTEXT.md](../MYSPCE_PROJECT_CONTEXT.md). Anything not confirmed there is marked *(proposed)*.

## 1. Inputs and constraints

| Item | Status | Source |
|---|---|---|
| Single user; Google + email/password login | Confirmed | project context §2 |
| Server-side AES encryption now, zero-knowledge later | Confirmed | project context §2 |
| Tailwind CSS v4, class-based dark mode | Confirmed | project context §2 |
| Mobile responsive first | Confirmed | project context §2 |
| Hosting: free AWS tier later | Confirmed | project context §2 |
| Screens: Dashboard, Notes, Credentials, Tasks & Goals, Settings, Admin | Confirmed (from the design) | project context §2 |
| NestJS + TypeORM + MySQL backend | Planned, not built | project context §3 |
| CSR SPA, no SSR in v1 | Assumption A1 | project context §5 |
| Token storage: memory + httpOnly cookie | Assumption A2 | project context §5 |
| Roles `super_admin/admin/user` | Assumption A3 | project context §5 |
| Tiptap editor | Assumption A5 | project context §5 |

Open questions to settle **before** milestone M3: final API shapes, the note body field and format, category counts, admin-panel scope, 2FA. See project context §7.

## 2. Repository setup (day 1)

1. `ng new myspce --style=css --ssr=false` with the Angular 22 CLI. Accept the defaults: zoneless, Vitest, strict.
2. Add Tailwind v4 and the `@custom-variant dark (&:where(.dark, .dark *));` line.
3. Add ESLint (`ng add @angular-eslint/schematics`), Prettier, and a pre-commit hook (lint-staged).
4. Create the folder skeleton from [project context §6](../MYSPCE_PROJECT_CONTEXT.md#6-proposed-frontend-architecture).
5. Copy only the **reviewed** pieces from your lab repo: `core/http`, `core/auth`, shared UI kit, `src/testing`. Rewrite anything you'd do differently now.
6. CI (lint, test, build) from day 1, using Lab 14's `ci.yml`.

## 3. Milestones

Each milestone ends with a demo build, green CI, and a tag.

| # | Milestone | Scope | Depends on backend? | Exit criteria |
|---|---|---|---|---|
| M0 | Foundation | repo, CI, Tailwind, theme, shell layout (sidebar, top bar, mobile drawer), 404, UI kit (Button, Modal, Confirm, Toast, Menu, EmptyState, ErrorState, Skeleton, PageHeader, ChipFilter) | No | Lighthouse a11y ≥ 95 on the shell; UI kit tests pass |
| M1 | Auth | login (email/password), Google callback, session restore, refresh, logout, guards, idle lock *(proposed)* | Mock → real `/auth/*` | E2E: login, reload, logout; one refresh per expiry |
| M2 | Bootstrap & settings | `GET /bootstrap`, feature flags, nav from config, Settings: Profile, Appearance, Security (change password) | Mock → real | Theme persists; flags hide modules |
| M3 | Notes | list (URL state, search, chips, pagination), editor (defer-loaded), pin/favourite, delete → Trash + Undo, restore | Real `/notes` preferred | E2E: create → search → delete → undo |
| M4 | Credentials | list + detail, add/edit modal, reveal with re-auth, copy, auto-hide | Real `/credentials` + `/auth/reauth` required | Security checklist items for secrets all ✅ |
| M5 | Tasks & Goals | today/tomorrow/week/upcoming/completed views, optimistic toggle, create/edit, goals tab, habits placeholder (v1.1) | Real `/tasks` | E2E: toggle updates dashboard progress |
| M6 | Dashboard | summary tiles, today's tasks, recent activity, week progress; widget registry; `@defer` | Real `/dashboard/*` | Lighthouse perf ≥ 90 mobile |
| M7 | Global search & Trash | top-bar search (`GET /search`), Trash page with restore/purge *(purge proposed)* | Real | Keyboard-only search works |
| M8 | Admin panel | appearance, modules on/off, navigation order (scope to be confirmed) | Real admin APIs (to be designed) | Role `user` can't reach admin (UI and API) |
| M9 | Hardening & release | CSP enforcing, error handler, budgets, a11y audit, release checklist, deploy to AWS | Real | [Module 20 release checklist](../20-security-and-deployment/04-deployment-and-ci.md) fully ✅ |

## 4. Implementation order inside each feature

Follow the same order every time. It keeps features consistent and testable.

1. **Models** in `core/models`, matching the API contract exactly. No invented fields.
2. **API service** (`createCrudApi` + extras) + HTTP tests.
3. **Mock routes** (until the backend endpoint exists) that match the contract.
4. **Store** (signals, resources, mutations with optimistic update/rollback) + store tests.
5. **Routes** (lazy, guards, titles).
6. **Page component** with all four states (loading, empty, error, content).
7. **Presentational components** (cards, rows, forms) + component tests.
8. **Interactions** (dialogs, confirm, toasts, keyboard).
9. **E2E** for the feature's critical path.
10. **Review** against the code-review checklist in [Module 20 note 4](../20-security-and-deployment/04-deployment-and-ci.md#8-code-review-and-maintainability).

## 5. Definition of done (every PR)

- [ ] Matches the design on mobile (360 px), tablet and desktop; light and dark.
- [ ] Loading, empty, error and success states handled.
- [ ] Keyboard-operable; visible focus; labels/ARIA correct; axe clean.
- [ ] No `any`; inputs typed; no secrets or PII in logs.
- [ ] Unit/component tests for logic; E2E updated if the critical path changed.
- [ ] Bundle budgets pass; no new eager dependency without a reason.
- [ ] `MYSPCE_PROJECT_CONTEXT.md` updated if a decision or API changed.

## 6. Suggested timeline *(proposed — adjust to your hours)*

At ~10 focused hours a week, alongside backend work:

| Weeks | Milestones |
|---|---|
| 1–2 | M0 |
| 3 | M1 |
| 4 | M2 |
| 5–6 | M3 |
| 7–8 | M4 |
| 9 | M5 |
| 10 | M6 |
| 11 | M7 |
| 12–13 | M8 (after scope is confirmed) |
| 14 | M9 + first production deploy |

Backend milestones should land **one step ahead** of the frontend milestone that needs them. Until then, keep the mock in sync with the agreed contract.

## 7. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| API contract changes late | rework in models/stores | contract-first: agree DTOs (OpenAPI from NestJS *(proposed)*) before each milestone; models in one place |
| Secrets exposure in the UI | severe | reveal-only endpoint, auto-hide, no logs/toasts, CSP, dependency audits |
| Zero-knowledge later changes data flow | large refactor of credentials | keep all crypto-sensitive logic behind `CredentialsApi`/a `VaultService` boundary now |
| Admin panel scope creep | delays v1 | ship M8 with the three confirmed-in-principle areas; everything else v1.1 |
| Over-engineering state | slower delivery | signal stores first; revisit NgRx only if the pain is real |
| Mobile performance | poor UX on phones | Lab 13 budgets in CI from M0; test on a real mid-range phone each milestone |
| Solo developer burnout | stalled project | small milestones, demo builds, cut scope before cutting quality |

## 8. Tracking

Create one GitHub issue per milestone with the exit criteria as a checklist, and one PR per numbered step in §4. Update [PROGRESS_TRACKER.md](../PROGRESS_TRACKER.md) when a milestone closes.
